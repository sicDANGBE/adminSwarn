# Ansible Kubernetes Playbooks

Objectif principal: deployer et maintenir un cluster Kubernetes (K3s) et les applications d'administration.
Les anciens playbooks Swarm restent disponibles mais sont explicitement marques comme legacy.

**Structure**
- `site.yml`: orchestration principale (base systeme, k8s, apps k8s, runner)
- `inventory.yml`: inventaire et variables globales
- `roles/`: roles Ansible

**Execution recommandee (Kubernetes)**
- Base systeme: `ansible-playbook -i inventory.yml site.yml --tags common`
- Cluster K3s: `ansible-playbook -i inventory.yml site.yml --tags k8s`
- Apps K8s (admin): `ansible-playbook -i inventory.yml site.yml --tags app_k8s`
- Runner K8s: `ansible-playbook -i inventory.yml site.yml --tags k8s_runner`
- Tout en une fois (k8s): `ansible-playbook -i inventory.yml site.yml --tags common,k8s,app_k8s,k8s_runner`

**Legacy Swarm (desactive par defaut)**
- Docker: `ansible-playbook -i inventory.yml site.yml --tags docker`
- Swarm: `ansible-playbook -i inventory.yml site.yml --tags swarm`
- Apps Swarm: `ansible-playbook -i inventory.yml site.yml --tags swarm_apps`

**Securite / operations sensibles**
- Nettoyage destructif: `ansible-playbook -i inventory.yml site.yml --tags clean`
- Audit securite: `ansible-playbook -i inventory.yml site.yml --tags audit`

**Variables cles**
- Domaine des ingress: `main_domain` dans `inventory.yml`
- K3s: `k3s_version`, `k3s_flannel_iface` dans `roles/k8s/defaults/main.yml`
- StorageClass K8s: `storage_class` dans `roles/app_k8s/vars/main.yml`
- Secrets: `group_vars/all/secret.yml` (ex: `github_token`, `postgres_password`, `grafana_admin_password`)

**Notes**
- `site.yml` ne lance plus Docker/Swarm par defaut. Les roles legacy sont tagges `never`.
