# Πλήρης Οδηγός Setup — Medical Appointment Service DevOps

Στήνει τα πάντα από το μηδέν: 2 VMs, SSH access, Ansible, Jenkins, Kubernetes,
HTTPS/FQDN, και deploy της εφαρμογής.

**Όλα τα βήματα τρέχουν από ένα μέρος — το WSL σου.** Αν δεν έχεις `gcloud` εκεί:
```bash
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init
```

---

## Φάση 0: GCP Project + Billing

```bash
gcloud projects create devops-project-lambros-2026 --name="DevOps Project"
gcloud config set project devops-project-lambros-2026

gcloud billing accounts list
gcloud billing projects link devops-project-lambros-2026 \
  --billing-account=<ACCOUNT_ID>

gcloud services enable compute.googleapis.com
```
Επιβεβαίωση: `gcloud billing projects describe devops-project-lambros-2026` → `billingEnabled: true`

---

## Φάση 1: Δημιουργία VMs

```bash
gcloud compute instances create devops-vm \
  --zone=europe-west1-b --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud \
  --boot-disk-size=30GB

gcloud compute instances create k8s-vm \
  --zone=europe-west1-b --machine-type=e2-medium \
  --image-family=ubuntu-2204-lts --image-project=ubuntu-os-cloud \
  --boot-disk-size=30GB
```
```bash
gcloud compute instances list --format='table(name,networkInterfaces[0].accessConfigs[0].natIP)'
```
📝 Σημείωσέ τα ως `<VM1_IP>`, `<VM2_IP>`.

**Bonus, προτείνεται**: πάγωσε το IP του VM1 (γλιτώνει μελλοντικά CORS/hosts.yaml μπερδέματα):
```bash
gcloud compute addresses create devops-vm-ip --addresses=<VM1_IP> --region=europe-west1
```

---

## Φάση 2: Firewall rules

```bash
gcloud compute firewall-rules create allow-devops-ports \
  --allow=tcp:22,tcp:3000,tcp:8025 --description="Frontend, MailHog UI"

gcloud compute firewall-rules create allow-jenkins \
  --allow=tcp:8080 --description="Jenkins UI"

gcloud compute firewall-rules create allow-backend-8081 \
  --allow=tcp:8081 --description="Backend API (VM1)"

gcloud compute firewall-rules create allow-k8s-ports \
  --allow=tcp:80,tcp:443,tcp:16443,tcp:10250,tcp:10255 --description="Microk8s (VM2)"

gcloud compute firewall-rules create allow-https \
  --allow=tcp:80,tcp:443 --description="Caddy reverse proxy (HTTPS)"
```

---

## Φάση 3: SSH — προσωπικό key

```bash
gcloud compute ssh devops-vm --zone=europe-west1-b
```
📝 Πρόσεξε το username στο prompt (π.χ. `lambr@devops-vm:~$`) — αυτό είναι το `ansible_user`.
```bash
exit
gcloud compute ssh k8s-vm --zone=europe-west1-b
exit
```

---

## Φάση 4: SSH — dedicated `jenkins-deploy` key

```bash
ssh-keygen -t ed25519 -f ~/.ssh/jenkins-deploy-key -N "" -C "jenkins-deploy"
cat ~/.ssh/jenkins-deploy-key.pub
```
Πρόσθεσέ το σε **και τα δύο** VMs:
```bash
gcloud compute ssh devops-vm --zone=europe-west1-b
echo "PASTE_PUBLIC_KEY" >> ~/.ssh/authorized_keys
exit

gcloud compute ssh k8s-vm --zone=europe-west1-b
echo "PASTE_PUBLIC_KEY" >> ~/.ssh/authorized_keys
exit
```

---

## Φάση 5: `hosts.yaml`

```yaml
---
  appservers:
    hosts:
      appserver:
        ansible_host: <VM1_IP>
        ansible_user: <username>

  dbservers:
    hosts:
      appserver:

  k8sservers:
    hosts:
      k8snode:
        ansible_host: <VM2_IP>
        ansible_user: <username>
```
```bash
git add hosts.yaml && git commit -m "Update IPs" && git push
```

---

## Φάση 6: sudoers (και στα δύο VMs)

```bash
gcloud compute ssh devops-vm --zone=europe-west1-b
echo "<username> ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/<username>
exit

gcloud compute ssh k8s-vm --zone=europe-west1-b
echo "<username> ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/<username>
exit
```

---

## Φάση 7: Ansible playbooks

```bash
cd ~/projects/Ansible-DevOps-Project/ansible
eval $(ssh-agent)
ssh-add ~/.ssh/google_compute_engine

ansible-playbook -i hosts.yaml playbooks/docker.yaml      # VM1
ansible-playbook -i hosts.yaml playbooks/jenkins.yaml     # VM1
ansible-playbook -i hosts.yaml playbooks/microk8s.yaml    # VM2
```

**Microk8s permissions fix** (χρειάζεται χειροκίνητα μέσα στο VM2, μία φορά):
```bash
gcloud compute ssh k8s-vm --zone=europe-west1-b
sudo usermod -a -G microk8s <username>
sudo chown -R <username> ~/.kube
newgrp microk8s
kubectl get nodes   # επιβεβαίωση: Ready
exit
```

---

## Φάση 8: Jenkins UI

```bash
gcloud compute ssh devops-vm --zone=europe-west1-b
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Άνοιξε `http://<VM1_IP>:8080`:
1. Unlock με το password
2. Install suggested plugins **+ επιπλέον: SSH Agent plugin**
3. Φτιάξε admin user
4. **Credentials** (Manage Jenkins → Credentials → System → Global):
   - `ghcr` — Username/password (GitHub username + PAT scope `write:packages`)
   - `vm-deploy-key` — SSH Username with private key (username από Φάση 3, private key = `cat ~/.ssh/jenkins-deploy-key`)
5. **4 Jobs** (New Item → Pipeline → "Pipeline script from SCM"):

| Job | Repo | Branch | Script Path |
|---|---|---|---|
| `backend-job` | Medical_Appointment_Service.git | main | `backend/JenkinsFile` |
| `frontend-job` | Medical_Appointment_Service.git | main | `frontend/JenkinsFile` |
| `ansible` | Ansible-DevOps-Project.git | main | `Jenkinsfile` |
| `pipeline-orchestrator` | Medical_Appointment_Service.git | main | `JenkinsFile` |

6. `pipeline-orchestrator` → Configure → Build Triggers → ✅ "GitHub hook trigger for GITScm polling"

---

## Φάση 9: GitHub Webhook

GitHub repo (`Medical_Appointment_Service`) → **Settings → Webhooks → Add webhook**:
- Payload URL: `http://<VM1_IP>:8080/github-webhook/` (πρόσεξε το τελικό `/`, και το σωστό "github", όχι typo)
- Content type: `application/json`
- SSL verification: **Disable** (το Jenkins είναι σε HTTP, όχι HTTPS)
- Events: **Just the push event**

Επιβεβαίωση: "Recent Deliveries" → πράσινο ✅ στο ping.

*(Εναλλακτικά, αυτόματα: `GITHUB_TOKEN=ghp_xxx ansible-playbook -i hosts.yaml playbooks/github-webhook.yaml -e jenkins_public_ip=<VM1_IP>`)*

---

## Φάση 10: HTTPS/FQDN (Caddy + DuckDNS)

1. [duckdns.org](https://www.duckdns.org) → login → φτιάξε subdomain → update IP με το static IP του VM1
2. ```bash
   ansible-playbook -i hosts.yaml playbooks/https.yaml -e fqdn=το-domain-σου.duckdns.org
   ```
3. Ενημέρωσε `CORS_ALLOWED_ORIGINS` στο `docker-compose.yml.j2` με το νέο https domain
4. Redeploy: `ansible-playbook -i hosts.yaml playbooks/deploy.yaml`

---

## Φάση 11: Deploy

Jenkins UI → job `ansible` → **Build Now** (τρέχει `deploy.yaml` + `k8s-deploy.yaml`)

Ή ολόκληρη η αλυσίδα: job `pipeline-orchestrator` → **Build Now**

---

## Τελικός έλεγχος

- [ ] `https://το-domain-σου.duckdns.org` → φορτώνει, login δουλεύει (Docker/VM1)
- [ ] `http://<VM2_IP>` → φορτώνει, login δουλεύει (Kubernetes/VM2)
- [ ] `docker ps` στο VM1 → 4 containers `Up`
- [ ] `kubectl get pods -n medical-appointment` στο VM2 → 4 pods `Running`
- [ ] Jenkins `pipeline-orchestrator` → όλα τα stages πράσινα
- [ ] `git push` → αυτόματο trigger, χωρίς χειροκίνητο "Build Now"

## Πρόσβαση στο MailHog

- **Docker (VM1)**: `http://<VM1_IP>:8025` (απευθείας, ήδη ανοιχτό firewall)
- **Kubernetes (VM2)**: μέσω port-forward (ClusterIP, όχι δημόσιο):
  ```bash
  ssh -L 8025:localhost:8025 -i ~/.ssh/google_compute_engine <username>@<VM2_IP>
  # μέσα στο SSH session:
  kubectl port-forward svc/mailhog -n medical-appointment 8025:8025
  # μετά, browser: http://localhost:8025
  ```

---

## Troubleshooting — γνωστά προβλήματα

| Σύμπτωμα | Αιτία | Λύση |
|---|---|---|
| `docker: permission denied` στο Jenkins | `jenkins` user δεν ήταν στην ομάδα `docker` όταν ξεκίνησε το service | `jenkins.yaml` το κάνει + restart, ήδη automated |
| `Host key verification failed` | Μη-διαδραστικό SSH δεν επιβεβαιώνει host key | `ansible_ssh_common_args` στο `group_vars/all.yaml` |
| `address already in use :8080` | Jenkins + backend container στο ίδιο host port | Backend host port → 8081 |
| CORS 403 μετά από αλλαγή IP/domain | Hardcoded allowed origin | `CORS_ALLOWED_ORIGINS` env var, δυναμικό μέσω template |
| `Insufficient permissions to access MicroK8s` | group membership δεν ισχύει σε ενεργό session | `usermod -a -G microk8s` + `newgrp microk8s` |
| Webhook 403 "No valid crumb" | Λάθος path στο Payload URL (typo) | Έλεγξε ότι είναι ακριβώς `/github-webhook/` |
| `No package matching docker-ce` | Hardcoded/άγνωστο release codename | Dynamic `ansible_distribution_release` + fallback map |
| Apt lock held by άλλη διεργασία | `unattended-upgrades`/`apt-daily` τρέχει στο background σε φρέσκο VM | Stop τα σχετικά services πριν το apt install |
