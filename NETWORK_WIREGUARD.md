# Architecture Réseau WireGuard - Nœuds OVH vers Raspberry Pi en direct

## Objectif

Permettre à tous les nœuds OVH du cluster de joindre durablement la Raspberry Pi hors cluster sur `10.8.0.12`, sans WireGuard dans Kubernetes et sans hub intermédiaire.

## Architecture retenue

### Rôle des machines

- `ovh.core`
  - nœud OVH
  - peer WireGuard avec IP `10.8.0.11/24`

- `ovh.worker.01`
  - nœud OVH
  - peer WireGuard avec IP `10.8.0.21/24`

- `ovh.worker.02`
  - nœud OVH
  - peer WireGuard avec IP `10.8.0.22/24`

- `raspberry.media`
  - Raspberry Pi hors cluster
  - IP WireGuard `10.8.0.12/32`
  - peer WireGuard direct des trois nœuds OVH
  - API derrière Traefik Swarm, atteignable via `http://10.8.0.12/health/live` avec `Host: mediaplayer.local`

### Flux réseau

Chaque nœud OVH parle directement à la Raspberry via son propre tunnel WireGuard:

```text
ovh.core / ovh.worker.01 / ovh.worker.02
  -> tunnel WireGuard local
  -> Raspberry Pi 10.8.0.12
```

## Pourquoi cette version est la bonne

- pas de WireGuard dans Kubernetes
- pas de hub WireGuard intermédiaire
- pas de routage statique via `ovh.core`
- pas de dépendance à la topologie réseau publique OVH entre VPS
- chaque nœud OVH peut joindre directement `10.8.0.12`
- topologie simple à diagnostiquer avec `wg show` sur chaque machine

## Fichiers Ansible concernés

### Inventaire et variables

- `inventory.yml`
  - groupe `wireguard_mesh`
  - hôtes `ovh.core`, `ovh.worker.01`, `ovh.worker.02`, `raspberry.media`

- `group_vars/all/wireguard.yml`
  - topologie globale
  - IP WireGuard de la Raspberry et des nœuds OVH
  - URL de validation via Traefik Swarm Raspberry

- `host_vars/ovh.core.yml`
  - IP WireGuard `10.8.0.11`
  - endpoint public `151.80.147.175:51820`

- `host_vars/ovh.worker.01.yml`
  - IP WireGuard `10.8.0.21`
  - endpoint public `151.80.144.195:51820`

- `host_vars/ovh.worker.02.yml`
  - IP WireGuard `10.8.0.22`
  - endpoint public `151.80.146.40:51820`

- `group_vars/wireguard_ovh_peers/main.yml`
  - configuration commune des peers OVH
  - peer Raspberry avec `AllowedIPs = 10.8.0.12/32`
  - ouverture UDP 51820 sur l’interface publique des nœuds OVH

- `group_vars/wireguard_raspberry/main.yml`
  - configuration WireGuard de la Raspberry
  - trois peers OVH avec `PersistentKeepalive = 25`

- `group_vars/agents/network_cleanup.yml`
  - suppression de l’ancienne route statique `10.8.0.12/32` via `ovh.core`

### Rôles

- `roles/wireguard`
  - installe WireGuard
  - réutilise les clés existantes si elles sont déjà présentes
  - génère automatiquement une clé sur un nœud OVH si absente
  - calcule la clé publique pour les autres peers
  - rend `wg-media.conf`
  - recharge `wg-quick@wg-media`

- `roles/static_routes`
  - supprime proprement la route statique legacy sur les workers
  - désactive le service systemd de routes statiques si plus aucune route n’est définie

## Gestion des clés

Aucune génération manuelle n’est nécessaire.

Le rôle `wireguard` fait ceci:

- sur la Raspberry: relit la clé privée déjà présente dans `/etc/wireguard/wg-media.conf`
- sur chaque nœud OVH: réutilise une clé existante si présente, sinon en génère une automatiquement

Si tu veux imposer une clé sur un nœud précis, ajoute `wireguard_private_key` dans son `host_vars`.

## Déploiement

### 1. Vérifier l’inventaire

Le fichier [inventory.yml](/mnt/new_root/data/srv/microEntreprise/ansible/inventory.yml#L1) doit permettre à Ansible d’atteindre:

- `ovh.core`
- `ovh.worker.01`
- `ovh.worker.02`
- `rpi`

### 2. Déployer WireGuard

```bash
ansible-playbook -i inventory.yml site.yml --tags wireguard --ask-vault-pass
```

Ce run:

- lit la clé existante sur la Raspberry
- génère les clés manquantes sur les nœuds OVH
- écrit tous les `wg-media.conf`
- recharge `wg-quick@wg-media`
- supprime l’ancien routage statique via `ovh.core` sur les workers

### 3. Vérifier qu’il ne reste plus de route legacy

```bash
ansible -i inventory.yml agents -b -a "ip route show 10.8.0.12/32"
ansible -i inventory.yml agents -b -a "systemctl status ansible-static-routes --no-pager"
```

Résultat attendu:

- pas de route `10.8.0.12/32 via <ip_publique_ovh.core>`
- service `ansible-static-routes` absent ou désactivé

## Vérification

### Vérifier le tunnel WireGuard

```bash
ansible -i inventory.yml wireguard_ovh_peers -b -a "wg show"
ansible -i inventory.yml wireguard_raspberry -a "sudo wg show"
```

### Vérifier depuis tous les nœuds OVH

```bash
ansible -i inventory.yml wireguard_ovh_peers -b -a "ping -c 2 10.8.0.12"
ansible -i inventory.yml wireguard_ovh_peers -b -a "curl -fsS -H 'Host: mediaplayer.local' http://10.8.0.12/health/live"
```

### Vérifier la route effective

```bash
ansible -i inventory.yml wireguard_ovh_peers -b -a "ip route get 10.8.0.12"
```

Résultat attendu sur chaque nœud OVH:

- route vers `10.8.0.12` via `wg-media`
- plus de route via l’IP publique de `ovh.core`

## Diagnostic

### Nœuds OVH

```bash
ansible -i inventory.yml wireguard_ovh_peers -b -a "systemctl status wg-quick@wg-media --no-pager"
ansible -i inventory.yml wireguard_ovh_peers -b -a "journalctl -u wg-quick@wg-media -n 50 --no-pager"
ansible -i inventory.yml wireguard_ovh_peers -b -a "ip addr show wg-media"
ansible -i inventory.yml wireguard_ovh_peers -b -a "ip route show table main"
```

### Raspberry

```bash
ansible -i inventory.yml wireguard_raspberry -a "sudo cat /etc/wireguard/wg-media.conf"
ansible -i inventory.yml wireguard_raspberry -a "sudo wg show"
```

## Points d’attention

- chaque nœud OVH écoute en UDP `51820` sur son IP publique
- la Raspberry initie les sessions directes vers les nœuds OVH avec `PersistentKeepalive`
- `AllowedIPs` de la Raspberry couvrent `10.8.0.11/32`, `10.8.0.21/32` et `10.8.0.22/32`
- chaque nœud OVH a sa propre paire de clés WireGuard
- Kubernetes reste hors du périmètre VPN: on garantit seulement la connectivité IP depuis les nœuds
