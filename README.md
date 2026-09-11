# Ansible-DevOps-Project

Πλήρης αυτοματοποίηση εγκατάστασης/deployment του
[Medical_Appointment_Service](https://github.com/LamprosChalatsis/Medical_Appointment_Service)
στα 3 environments που ζητάει η εργασία: **plain VM**, **Docker**, **Kubernetes**.

---

## Αρχιτεκτονική Υποδομής

- **VM1** (`devops-vm`): Docker + Docker Compose + Jenkins — Docker environment & CI/CD
- **VM2** (`k8s-vm`): Microk8s — Kubernetes environment
- Δημόσια πρόσβαση: HTTPS/FQDN μέσω Caddy (VM1) + DuckDNS

## Δομή

```
ansible/
├── ansible.cfg
├── hosts.yaml                       # inventory: appservers, dbservers, k8sservers
├── group_vars/
│   ├── all.yaml                      # ansible_ssh_common_args, python interpreter
│   └── dbservers.yaml                # db credentials
├── k8s/                               # Kubernetes manifests
│   ├── 00-namespace.yaml
│   ├── 01-mailhog.yaml
│   ├── 02-secrets-config.yaml         # Secret (db creds) + ConfigMap (CORS, mail)
│   ├── 03-db.yaml                     # PVC + Deployment + Service
│   ├── 04-backend.yaml
│   ├── 05-frontend.yaml
│   └── 06-ingress.yaml
├── playbooks/
│   ├── docker.yaml                    # [Docker env]    εγκατάσταση Docker Engine
│   ├── jenkins.yaml                   # [CI/CD]         Jenkins + build tools (Maven, Node 20, Git)
│   ├── deploy.yaml                    # [Docker env]    docker compose pull + up
│   ├── microk8s.yaml                  # [K8s env]       εγκατάσταση Microk8s + addons
│   ├── k8s-deploy.yaml                # [K8s env]       αντιγραφή manifests + kubectl apply
│   ├── https.yaml                     # Caddy reverse proxy, αυτόματο HTTPS
│   ├── postgres.yaml                  # [Plain VM env]  bare-metal PostgreSQL
│   ├── backend-vm.yaml                # [Plain VM env]  backend ως systemd service
│   └── frontend-vm.yaml               # [Plain VM env]  frontend με nginx
├── templates/
│   ├── docker-compose.yml.j2
│   ├── backend.service.j2
│   └── nginx-frontend.conf.j2
└── Jenkinsfile                        # Deploy stages: Docker (VM1) + Kubernetes (VM2)
```

## Προαπαιτούμενα

- Ansible τοπικά (`sudo apt install ansible`)
- `gcloud` CLI, συνδεδεμένο σε ενεργό GCP project με billing
- SSH: προσωπικό key (μέσω `gcloud compute ssh`) + dedicated `jenkins-deploy-key`
  (χρησιμοποιείται από Jenkins μέσω SSH Agent, όχι hardcoded στο filesystem)

## CI/CD — Jenkins × Ansible

Το `Jenkinsfile` αυτού του repo τρέχει ως το Jenkins job `ansible`, με δύο στάδια:

```groovy
stage('Deploy to Docker (VM1)')      → ansible-playbook playbooks/deploy.yaml
stage('Deploy to Kubernetes (VM2)')  → ansible-playbook playbooks/k8s-deploy.yaml
```

Τα SSH credentials παρέχονται στο Jenkins μέσω **SSH Agent plugin** +
**Jenkins Credentials Store** (`vm-deploy-key`) — όχι μέσω static key file στο
filesystem, για λόγους ασφαλείας (dedicated key, όχι το προσωπικό του developer).

## Kubernetes — τι δείχνει

- **Namespace** για οργάνωση resources
- **Deployments + Services** για κάθε component (self-healing, service discovery)
- **PersistentVolumeClaim** για τη βάση (δεδομένα επιβιώνουν σε restart του pod)
- **Secret** (db credentials) + **ConfigMap** (mail/CORS config) — αντί για
  hardcoded env vars
- **Ingress** (nginx controller, μέσω Microk8s `ingress` addon) για routing
  εξωτερικής κίνησης

## Troubleshooting — γνωστά προβλήματα & λύσεις

| Σύμπτωμα | Αιτία | Λύση |
|---|---|---|
| `docker: permission denied` στο Jenkins | Ο `jenkins` user δεν ήταν στην ομάδα `docker` όταν το service ξεκίνησε | `jenkins.yaml` το κάνει αυτόματα + restart service |
| `Host key verification failed` σε automated SSH | Ο μη-διαδραστικός χρήστης δεν έχει pre-trusted το target host | `ansible_ssh_common_args` με `StrictHostKeyChecking=no` στο `group_vars/all.yaml` |
| `address already in use` στο port 8080 | Jenkins και backend container στο ίδιο VM διεκδικούν το ίδιο host port | Backend host port μετακινήθηκε σε 8081 |
| CORS 403 μετά από αλλαγή IP/domain | Hardcoded allowed origin στο Spring Security config | `CORS_ALLOWED_ORIGINS` env var, ρυθμίζεται δυναμικά από το Ansible template |

## Σχετικά repos

- [Medical_Appointment_Service](https://github.com/LamprosChalatsis/Medical_Appointment_Service) — η εφαρμογή
