# Export de projet

_Généré le 2025-12-04T22:56:15+01:00_

## deploy.yml

```yaml
---
- name: Déploiement de la Stack d'Administration (Manager)
  hosts: ovh.core
  become: yes
  roles:
    - deploy
```

## inventory.yml

```yaml
all:
  children:
    # Groupe global
    swarm_nodes:
      children:
        manager:
          hosts:
            ovh.core:
              ansible_host: vps-940a692a.vps.ovh.net
        workers:
          hosts:
            ovh.worker.01:
              ansible_host: vps-255dab72.vps.ovh.net
            ovh.worker.02:
              ansible_host: vps-9e3ed523.vps.ovh.net
      vars:
        sysadmin_user: supadmin
        # Le manager est maintenant défini dynamiquement par le groupe
        manager_host: "{{ groups['manager'][0] }}"
        main_domain: dgsynthex.online
        
        # --- CONFIGURATION CONNEXION ---
        ansible_user: supadmin
        ansible_port: 49281
        ansible_python_interpreter: /usr/bin/python3
        ansible_become: yes
        local_public_key: "~/.ssh/id_rsa.pub"
        docker_hub_user: spadmdck
        docker_hub_pat: "Irmapil-1989"
```

## pb/audit_security.yml

```yaml
---
- name: Audit de Sécurité et Configuration
  hosts: swarm_nodes
  become: yes
  gather_facts: yes
  vars:
    report_path: "./reports"

  tasks:
    # --- 1. ETAT DU SYSTEME ---
    - name: Vérification de l'OS et Kernel
      command: uname -a
      register: sys_kernel

    - name: Liste des utilisateurs critiques (passwd)
      shell: grep -E '^(root|supadmin|dockremap|debian)' /etc/passwd
      register: sys_users

    # --- 2. DOCKER (Le point critique actuel) ---
    - name: Lecture du daemon.json
      command: cat /etc/docker/daemon.json
      register: docker_conf
      ignore_errors: yes

    - name: Vérification des fichiers de mapping (subuid)
      command: cat /etc/subuid
      register: subuid_file
      ignore_errors: yes

    - name: Statut du Service Docker
      command: systemctl status docker --no-pager
      register: docker_status
      ignore_errors: yes

    - name: Logs récents de Docker (si erreur)
      shell: journalctl -u docker -n 20 --no-pager
      register: docker_logs
      ignore_errors: yes

    - name: Vérification des permissions dossier Docker
      shell: ls -ld /var/lib/docker
      register: docker_perm
      ignore_errors: yes

    # --- 3. RESEAU & SECURITE ---
    - name: Règles NFTables actives
      command: nft list ruleset
      register: nft_rules
      ignore_errors: yes

    - name: Ports en écoute (SS)
      shell: ss -tulnp
      register: net_ports

    - name: Configuration SSH active (sans commentaires)
      shell: grep -vE '^#|^$' /etc/ssh/sshd_config
      register: ssh_conf

    # --- 4. GENERATION DU RAPPORT LOCAL ---
    
    - name: Création du dossier de rapports local
      delegate_to: localhost
      become: no
      file:
        path: "{{ report_path }}"
        state: directory

    - name: Écriture du rapport localement
      delegate_to: localhost
      become: no
      copy:
        dest: "{{ report_path }}/report_{{ inventory_hostname }}.txt"
        content: |
          =============================================
          RAPPORT D'AUDIT : {{ inventory_hostname }}
          Date : {{ ansible_date_time.iso8601 }}
          IP : {{ ansible_default_ipv4.address }}
          =============================================

          [1] NOYAU & OS
          ---------------------------------------------
          {{ sys_kernel.stdout }}

          [2] UTILISATEURS CLES
          ---------------------------------------------
          {{ sys_users.stdout }}

          [3] DOCKER : CONFIGURATION
          ---------------------------------------------
          >> daemon.json :
          {{ docker_conf.stdout | default('Fichier introuvable') }}

          >> subuid mapping :
          {{ subuid_file.stdout | default('Fichier introuvable') }}
          
          >> Permissions /var/lib/docker :
          {{ docker_perm.stdout | default('Dossier introuvable') }}

          [4] DOCKER : SANTE
          ---------------------------------------------
          >> Status SystemD :
          {{ docker_status.stdout | default('Service introuvable') }}

          >> Derniers Logs (Erreurs potentielles) :
          {{ docker_logs.stdout }}

          [5] PARE-FEU (NFTABLES)
          ---------------------------------------------
          {{ nft_rules.stdout | default('Aucune règle chargée') }}

          [6] PORTS OUVERTS
          ---------------------------------------------
          {{ net_ports.stdout }}

          [7] CONFIG SSH
          ---------------------------------------------
          {{ ssh_conf.stdout }}
```

## pb/configure_firewall_swarm.yml

```yaml
---
- name: Configuration et Validation du Pare-feu (Swarm + Web)
  hosts: swarm_nodes
  become: yes
  vars:
    # Récupération dynamique des IPs
    cluster_ips: "{{ groups['swarm_nodes'] | map('extract', hostvars, ['ansible_default_ipv4', 'address']) | join(', ') }}"
    # On définit explicitement qui est le Manager pour le ciblage des ports Web
    manager_host: "ovh.core" 

  tasks:
    # --- 1. CONFIGURATION DU PARE-FEU ---
    
    - name: Mise à jour de /etc/nftables.conf
      copy:
        dest: /etc/nftables.conf
        validate: '/usr/sbin/nft -c -f %s'
        content: |
          #!/usr/sbin/nft -f
          flush ruleset
          
          # Définition des IPs de confiance (le cluster)
          define SWARM_NODES = { {{ cluster_ips }} }

          table inet filter {
            chain input {
              type filter hook input priority 0; policy drop;

              # Base
              iifname "lo" accept
              ct state established,related accept
              ip protocol icmp accept
              ip6 nexthdr icmpv6 accept

              # SSH Admin (Port variable Ansible)
              tcp dport {{ ansible_port }} accept

              # ACCES WEB (HTTP/HTTPS)
              # Uniquement sur le Manager, car Traefik est en "mode: host" sur ce noeud
              {% if inventory_hostname == manager_host %}
              tcp dport { 80, 443 } accept
              {% endif %}

              # SWARM (Only from Cluster Nodes)
              ip saddr $SWARM_NODES tcp dport { 2377, 7946 } accept
              ip saddr $SWARM_NODES udp dport { 7946, 4789 } accept
              
              # Protocol ESP (Encryption Overlay)
              ip protocol esp accept
            }
            chain forward {
              type filter hook forward priority 0; policy drop;
              ct state established,related accept
              
              # DOCKER LOCAL
              iifname "docker0" accept
              oifname "docker0" accept

              # DOCKER SWARM OVERLAY
              # Nécessaire pour la communication inter-conteneurs et le routing mesh
              iifname "docker_gwbridge" accept
              oifname "docker_gwbridge" accept
            }
            chain output {
              type filter hook output priority 0; policy accept;
            }
          }
        mode: '0755'
      notify: Reload Nftables

    - name: Application immédiate du pare-feu
      meta: flush_handlers

    # --- 2. VALIDATION SSH (Canari local) ---

    - name: Vérification que SSH est toujours vivant
      wait_for_connection:
        timeout: 10
    
    # --- 3. VALIDATION CROISÉE SWARM (Le vrai test) ---

    - name: Installation de netcat (nécessaire pour le test)
      apt:
        pkg: netcat-openbsd
        state: present

    # ETAPE A : On ouvre une fausse écoute sur le Manager (Port 2377)
    # Option -k pour garder le port ouvert pour tous les workers
    - name: Démarrage simulation Swarm sur le Manager
      command: "timeout 25s nc -lk -p 2377"
      async: 30
      poll: 0
      when: inventory_hostname == manager_host
      register: swarm_listener

    # ETAPE B : Les Workers essaient de toucher le Manager
    - name: Test de connexion Swarm (Worker -> Manager)
      wait_for:
        host: "{{ hostvars[manager_host]['ansible_default_ipv4']['address'] }}"
        port: 2377
        timeout: 10
        state: started
      when: inventory_hostname != manager_host

    # ETAPE C : Nettoyage
    - name: Arrêt de la simulation sur le Manager
      shell: "pkill -f 'nc -lk -p 2377'"
      ignore_errors: yes
      when: inventory_hostname == manager_host

    - name: Succès
      debug:
        msg: "Validation RÉUSSIE : Pare-feu configuré (Web sur Manager, Swarm interne OK)."
      run_once: true

  handlers:
    - name: Reload Nftables
      service: name=nftables state=reloaded
```

## pb/init_swarm.yml

```yaml
---
# --- ETAPE 1 : INSTALLATION DES PRÉREQUIS (Sur tous les noeuds) ---
- name: Installation du SDK Python Docker
  hosts: swarm_nodes
  become: yes
  tasks:
    - name: Installation de python3-docker
      apt:
        pkg:
          - python3-docker # Le SDK officiel via APT (Recommandé sur Debian 12)
          - python3-pip    # Toujours utile
        state: present
        update_cache: yes

# --- ETAPE 2 : INITIALISATION DU MANAGER ---
- name: Initialisation du Manager (ovh.core)
  hosts: ovh.core
  become: yes
  tasks:
    - name: Initialiser le Swarm
      community.docker.docker_swarm:
        state: present
        # advertise_addr est crucial pour que les workers sachent qui contacter
        advertise_addr: "{{ ansible_default_ipv4.address }}"
      register: swarm_info

    - name: Récupération du Token Worker
      set_fact:
        worker_token: "{{ swarm_info.swarm_facts.JoinTokens.Worker }}"

# --- ETAPE 3 : JONCTION DES WORKERS ---
- name: Jonction des Workers au Cluster
  hosts: ovh.worker.01, ovh.worker.02
  become: yes
  vars:
    # On récupère l'IP et le Token depuis les "facts" du Manager
    manager_ip: "{{ hostvars['ovh.core']['ansible_default_ipv4']['address'] }}"
    token: "{{ hostvars['ovh.core']['worker_token'] }}"
  tasks:
    - name: Rejoindre le Swarm en tant que Worker
      community.docker.docker_swarm:
        state: join
        join_token: "{{ token }}"
        remote_addrs: [ "{{ manager_ip }}:2377" ]
```

## pb/reports/report_ovh.core.txt

```text
=============================================
RAPPORT D'AUDIT : ovh.core
Date : 2025-12-02T18:31:36Z
IP : 151.80.147.175
=============================================

[1] NOYAU & OS
---------------------------------------------
Linux vps-940a692a 6.1.0-40-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.153-1 (2025-09-20) x86_64 GNU/Linux

[2] UTILISATEURS CLES
---------------------------------------------
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian:/home/debian:/bin/bash
supadmin:x:1001:1001::/home/supadmin:/bin/bash
dockremap:x:999:994::/home/dockremap:/usr/sbin/nologin

[3] DOCKER : CONFIGURATION
---------------------------------------------
>> daemon.json :
{
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true
}

>> subuid mapping :
debian:100000:65536
supadmin:165536:65536
dockremap:165536:65536

>> Permissions /var/lib/docker :
drwx--x--- 3 root 165536 4096 Dec  2 18:20 /var/lib/docker

[4] DOCKER : SANTE
---------------------------------------------
>> Status SystemD :
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-12-02 18:20:38 UTC; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 18193 (dockerd)
      Tasks: 11
     Memory: 44.3M
        CPU: 7.426s
     CGroup: /system.slice/docker.service
             └─18193 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.818570238Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=cef5a1a49839 ep=gateway_ingress-sbox net=docker_gwbridge nid=dbf523416848
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.829682237Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=85050cbf4cea ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665421496Z" level=info msg="worker qmpr8hijiu95mafd88xinwgen was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665928330Z" level=info msg="worker kohp6n6zegz6r1hnkcuhsoew1 was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682129891Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682266493Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683205218Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683286093Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:26:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:26:03.253409850Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:31:03.453096669Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"

>> Derniers Logs (Erreurs potentielles) :
Dec 02 18:21:02 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:02.640447053Z" level=info msg="dispatcher starting" module=dispatcher node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.154768701Z" level=info msg="manager selected by agent for new session: { }" module=node/agent node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.154978494Z" level=info msg="waiting 0s before registering session" module=node/agent node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.247533103Z" level=info msg="worker vlflk1nflcz8dj3ok7opa91vm was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.250978922Z" level=info msg="Initializing Libnetwork Agent" advertise-addr=151.80.147.175 data-path-addr= listen-addr=0.0.0.0 local-addr=151.80.147.175 network-control-plane-mtu=1500 remote-addr-list="[]"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.251185482Z" level=info msg="New memberlist node - Node:vps-940a692a will use memberlist nodeID:535ca91ded85 with config:&{NodeID:535ca91ded85 Hostname:vps-940a692a BindAddr:0.0.0.0 AdvertiseAddr:151.80.147.175 BindPort:0 Keys:[[34 43 73 188 29 234 85 2 42 207 143 77 148 130 194 109] [110 234 157 124 18 188 214 97 80 123 34 88 86 113 168 29] [165 15 103 40 8 128 158 217 104 152 213 245 92 110 196 169]] PacketBufferSize:1400 reapEntryInterval:1800000000000 reapNetworkInterval:1825000000000 rejoinClusterDuration:10000000000 rejoinClusterInterval:60000000000 StatsPrintPeriod:5m0s HealthPrintPeriod:1m0s}"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.251163822Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.252069002Z" level=info msg="Node 535ca91ded85/151.80.147.175, joined gossip cluster"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.252253772Z" level=info msg="Node 535ca91ded85/151.80.147.175, added to nodes list"
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.256792551Z" level=info msg="cluster update event" module=dispatcher node.id=vlflk1nflcz8dj3ok7opa91vm
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.818570238Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=cef5a1a49839 ep=gateway_ingress-sbox net=docker_gwbridge nid=dbf523416848
Dec 02 18:21:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:03.829682237Z" level=info msg="sbJoin: gwep4 ''->'cef5a1a49839', gwep6 ''->''" eid=85050cbf4cea ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665421496Z" level=info msg="worker qmpr8hijiu95mafd88xinwgen was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.665928330Z" level=info msg="worker kohp6n6zegz6r1hnkcuhsoew1 was successfully registered" method="(*Dispatcher).register"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682129891Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.682266493Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683205218Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-940a692a dockerd[18193]: time="2025-12-02T18:21:05.683286093Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:26:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:26:03.253409850Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:03 vps-940a692a dockerd[18193]: time="2025-12-02T18:31:03.453096669Z" level=info msg="NetworkDB stats vps-940a692a(535ca91ded85) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"

[5] PARE-FEU (NFTABLES)
---------------------------------------------
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iifname "lo" accept
		ct state established,related accept
		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept
		tcp dport 49281 accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } tcp dport { 2377, 7946 } accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } udp dport { 4789, 7946 } accept
		ip protocol esp accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iifname "docker0" accept
		oifname "docker0" accept
		iifname "docker_gwbridge" accept
		oifname "docker_gwbridge" accept
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

[6] PORTS OUVERTS
---------------------------------------------
Netid State  Recv-Q Send-Q       Local Address:Port  Peer Address:PortProcess                                   
udp   UNCONN 0      0                  0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=11))
udp   UNCONN 0      0               127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=20))
udp   UNCONN 0      0            127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=18))
udp   UNCONN 0      0      151.80.147.175%ens3:68         0.0.0.0:*    users:(("systemd-network",pid=477,fd=10))
udp   UNCONN 0      0                  0.0.0.0:4789       0.0.0.0:*                                             
udp   UNCONN 0      0                     [::]:5355          [::]:*    users:(("systemd-resolve",pid=450,fd=13))
udp   UNCONN 0      0                        *:7946             *:*    users:(("dockerd",pid=18193,fd=34))      
tcp   LISTEN 0      4096            127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=21))
tcp   LISTEN 0      4096               0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=12))
tcp   LISTEN 0      4096         127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=450,fd=19))
tcp   LISTEN 0      128                0.0.0.0:49281      0.0.0.0:*    users:(("sshd",pid=13523,fd=3))          
tcp   LISTEN 0      4096                     *:7946             *:*    users:(("dockerd",pid=18193,fd=32))      
tcp   LISTEN 0      4096                     *:2377             *:*    users:(("dockerd",pid=18193,fd=27))      
tcp   LISTEN 0      4096                  [::]:5355          [::]:*    users:(("systemd-resolve",pid=450,fd=14))
tcp   LISTEN 0      128                   [::]:49281         [::]:*    users:(("sshd",pid=13523,fd=4))          

[7] CONFIG SSH
---------------------------------------------
Port 49281
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
ClientAliveInterval 120
```

## pb/reports/report_ovh.worker.01.txt

```text
=============================================
RAPPORT D'AUDIT : ovh.worker.01
Date : 2025-12-02T18:31:36Z
IP : 151.80.144.195
=============================================

[1] NOYAU & OS
---------------------------------------------
Linux vps-255dab72 6.1.0-40-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.153-1 (2025-09-20) x86_64 GNU/Linux

[2] UTILISATEURS CLES
---------------------------------------------
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian:/home/debian:/bin/bash
supadmin:x:1001:1001::/home/supadmin:/bin/bash
dockremap:x:999:994::/home/dockremap:/usr/sbin/nologin

[3] DOCKER : CONFIGURATION
---------------------------------------------
>> daemon.json :
{
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true
}

>> subuid mapping :
debian:100000:65536
supadmin:165536:65536
dockremap:165536:65536

>> Permissions /var/lib/docker :
drwx--x--- 3 root 165536 4096 Dec  2 18:20 /var/lib/docker

[4] DOCKER : SANTE
---------------------------------------------
>> Status SystemD :
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-12-02 18:20:37 UTC; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 18092 (dockerd)
      Tasks: 11
     Memory: 36.9M
        CPU: 5.247s
     CGroup: /system.slice/docker.service
             └─18092 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.691644130Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=60ddb6db40e2 ep=gateway_ingress-sbox net=docker_gwbridge nid=c0f4c74b9ddd
Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.703322155Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=a3f1990d8df0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:10.153756722Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:23:35.682745850Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:25:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:25:35.681935285Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:26:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:26:05.680283289Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:29:35.681878714Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.680737396Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.682496041Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:35.686172637Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"

>> Derniers Logs (Erreurs potentielles) :
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.671442892Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=qmpr8hijiu95mafd88xinwgen
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.677499893Z" level=info msg="Initializing Libnetwork Agent" advertise-addr=151.80.144.195 data-path-addr= listen-addr=0.0.0.0 local-addr=151.80.144.195 network-control-plane-mtu=1500 remote-addr-list="[151.80.147.175]"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.677752901Z" level=info msg="New memberlist node - Node:vps-255dab72 will use memberlist nodeID:ed3a6b16b0ce with config:&{NodeID:ed3a6b16b0ce Hostname:vps-255dab72 BindAddr:0.0.0.0 AdvertiseAddr:151.80.144.195 BindPort:0 Keys:[[34 43 73 188 29 234 85 2 42 207 143 77 148 130 194 109] [110 234 157 124 18 188 214 97 80 123 34 88 86 113 168 29] [165 15 103 40 8 128 158 217 104 152 213 245 92 110 196 169]] PacketBufferSize:1400 reapEntryInterval:1800000000000 reapNetworkInterval:1825000000000 rejoinClusterDuration:10000000000 rejoinClusterInterval:60000000000 StatsPrintPeriod:5m0s HealthPrintPeriod:1m0s}"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.679137537Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.679262362Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.679970038Z" level=info msg="The new bootstrap node list is:[151.80.147.175]"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.683252555Z" level=info msg="Node 535ca91ded85/151.80.147.175, joined gossip cluster"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.683323656Z" level=info msg="Node 535ca91ded85/151.80.147.175, added to nodes list"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.855151671Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:05.855283187Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.691644130Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=60ddb6db40e2 ep=gateway_ingress-sbox net=docker_gwbridge nid=c0f4c74b9ddd
Dec 02 18:21:06 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:06.703322155Z" level=info msg="sbJoin: gwep4 ''->'60ddb6db40e2', gwep6 ''->''" eid=a3f1990d8df0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-255dab72 dockerd[18092]: time="2025-12-02T18:21:10.153756722Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:23:35.682745850Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:25:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:25:35.681935285Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:26:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:26:05.680283289Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:29:35.681878714Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.680737396Z" level=info msg="NetworkDB stats vps-255dab72(ed3a6b16b0ce) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:05.682496041Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"
Dec 02 18:31:35 vps-255dab72 dockerd[18092]: time="2025-12-02T18:31:35.686172637Z" level=error msg="Bulk sync to node 65de0ea72bfb timed out"

[5] PARE-FEU (NFTABLES)
---------------------------------------------
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iifname "lo" accept
		ct state established,related accept
		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept
		tcp dport 49281 accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } tcp dport { 2377, 7946 } accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } udp dport { 4789, 7946 } accept
		ip protocol esp accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iifname "docker0" accept
		oifname "docker0" accept
		iifname "docker_gwbridge" accept
		oifname "docker_gwbridge" accept
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

[6] PORTS OUVERTS
---------------------------------------------
Netid State  Recv-Q Send-Q       Local Address:Port  Peer Address:PortProcess                                   
udp   UNCONN 0      0               127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=20))
udp   UNCONN 0      0            127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=18))
udp   UNCONN 0      0      151.80.144.195%ens3:68         0.0.0.0:*    users:(("systemd-network",pid=478,fd=10))
udp   UNCONN 0      0                  0.0.0.0:4789       0.0.0.0:*                                             
udp   UNCONN 0      0                  0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=11))
udp   UNCONN 0      0                        *:7946             *:*    users:(("dockerd",pid=18092,fd=28))      
udp   UNCONN 0      0                     [::]:5355          [::]:*    users:(("systemd-resolve",pid=433,fd=13))
tcp   LISTEN 0      4096         127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=19))
tcp   LISTEN 0      4096            127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=21))
tcp   LISTEN 0      128                0.0.0.0:49281      0.0.0.0:*    users:(("sshd",pid=13712,fd=3))          
tcp   LISTEN 0      4096               0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=433,fd=12))
tcp   LISTEN 0      4096                     *:7946             *:*    users:(("dockerd",pid=18092,fd=27))      
tcp   LISTEN 0      128                   [::]:49281         [::]:*    users:(("sshd",pid=13712,fd=4))          
tcp   LISTEN 0      4096                  [::]:5355          [::]:*    users:(("systemd-resolve",pid=433,fd=14))

[7] CONFIG SSH
---------------------------------------------
Port 49281
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
ClientAliveInterval 120
```

## pb/reports/report_ovh.worker.02.txt

```text
=============================================
RAPPORT D'AUDIT : ovh.worker.02
Date : 2025-12-02T18:31:36Z
IP : 151.80.146.40
=============================================

[1] NOYAU & OS
---------------------------------------------
Linux vps-9e3ed523 6.1.0-40-cloud-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.153-1 (2025-09-20) x86_64 GNU/Linux

[2] UTILISATEURS CLES
---------------------------------------------
root:x:0:0:root:/root:/bin/bash
debian:x:1000:1000:Debian:/home/debian:/bin/bash
supadmin:x:1001:1001::/home/supadmin:/bin/bash
dockremap:x:999:994::/home/dockremap:/usr/sbin/nologin

[3] DOCKER : CONFIGURATION
---------------------------------------------
>> daemon.json :
{
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "no-new-privileges": true
}

>> subuid mapping :
debian:100000:65536
supadmin:165536:65536
dockremap:165536:65536

>> Permissions /var/lib/docker :
drwx--x--- 3 root 165536 4096 Dec  2 18:20 /var/lib/docker

[4] DOCKER : SANTE
---------------------------------------------
>> Status SystemD :
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-12-02 18:20:38 UTC; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 18407 (dockerd)
      Tasks: 11
     Memory: 37.7M
        CPU: 5.456s
     CGroup: /system.slice/docker.service
             └─18407 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.709409551Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=d4bb76314650 ep=gateway_ingress-sbox net=docker_gwbridge nid=fc4e028e84b8
Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.721517407Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=e0a32fdf5ba0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:10.169551923Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:23:35.681905808Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:25:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:25:35.682903222Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:26:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:26:05.680074323Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:29:35.681648880Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.680853183Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.682300403Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:35.685523484Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"

>> Derniers Logs (Erreurs potentielles) :
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.671076421Z" level=info msg="initialized VXLAN UDP port to 4789 " module=node node.id=kohp6n6zegz6r1hnkcuhsoew1
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.677328290Z" level=info msg="Initializing Libnetwork Agent" advertise-addr=151.80.146.40 data-path-addr= listen-addr=0.0.0.0 local-addr=151.80.146.40 network-control-plane-mtu=1500 remote-addr-list="[151.80.147.175]"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.677606879Z" level=info msg="New memberlist node - Node:vps-9e3ed523 will use memberlist nodeID:65de0ea72bfb with config:&{NodeID:65de0ea72bfb Hostname:vps-9e3ed523 BindAddr:0.0.0.0 AdvertiseAddr:151.80.146.40 BindPort:0 Keys:[[34 43 73 188 29 234 85 2 42 207 143 77 148 130 194 109] [110 234 157 124 18 188 214 97 80 123 34 88 86 113 168 29] [165 15 103 40 8 128 158 217 104 152 213 245 92 110 196 169]] PacketBufferSize:1400 reapEntryInterval:1800000000000 reapNetworkInterval:1825000000000 rejoinClusterDuration:10000000000 rejoinClusterInterval:60000000000 StatsPrintPeriod:5m0s HealthPrintPeriod:1m0s}"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.679531154Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, joined gossip cluster"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.679680501Z" level=info msg="Node 65de0ea72bfb/151.80.146.40, added to nodes list"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.680708194Z" level=info msg="The new bootstrap node list is:[151.80.147.175]"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683716227Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, joined gossip cluster"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683762334Z" level=info msg="Node ed3a6b16b0ce/151.80.144.195, added to nodes list"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683789072Z" level=info msg="Node 535ca91ded85/151.80.147.175, joined gossip cluster"
Dec 02 18:21:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:05.683815623Z" level=info msg="Node 535ca91ded85/151.80.147.175, added to nodes list"
Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.709409551Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=d4bb76314650 ep=gateway_ingress-sbox net=docker_gwbridge nid=fc4e028e84b8
Dec 02 18:21:06 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:06.721517407Z" level=info msg="sbJoin: gwep4 ''->'d4bb76314650', gwep6 ''->''" eid=e0a32fdf5ba0 ep=ingress-endpoint net=ingress nid=3gu1hsyme8x8
Dec 02 18:21:10 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:21:10.169551923Z" level=error msg="Handler for GET /v1.52/nodes returned error: This node is not a swarm manager. Worker nodes can't be used to view or modify cluster state. Please run this command on a manager node or promote the current node to a manager."
Dec 02 18:23:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:23:35.681905808Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:25:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:25:35.682903222Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:26:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:26:05.680074323Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:29:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:29:35.681648880Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.680853183Z" level=info msg="NetworkDB stats vps-9e3ed523(65de0ea72bfb) - netID:3gu1hsyme8x889rtytxgqm3lr leaving:false netPeers:3 entries:3 Queue qLen:0+0 netMsg/s:0"
Dec 02 18:31:05 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:05.682300403Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"
Dec 02 18:31:35 vps-9e3ed523 dockerd[18407]: time="2025-12-02T18:31:35.685523484Z" level=error msg="Bulk sync to node ed3a6b16b0ce timed out"

[5] PARE-FEU (NFTABLES)
---------------------------------------------
table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		iifname "lo" accept
		ct state established,related accept
		ip protocol icmp accept
		ip6 nexthdr ipv6-icmp accept
		tcp dport 49281 accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } tcp dport { 2377, 7946 } accept
		ip saddr { 151.80.144.195, 151.80.146.40, 151.80.147.175 } udp dport { 4789, 7946 } accept
		ip protocol esp accept
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state established,related accept
		iifname "docker0" accept
		oifname "docker0" accept
		iifname "docker_gwbridge" accept
		oifname "docker_gwbridge" accept
	}

	chain output {
		type filter hook output priority filter; policy accept;
	}
}

[6] PORTS OUVERTS
---------------------------------------------
Netid State  Recv-Q Send-Q      Local Address:Port  Peer Address:PortProcess                                   
udp   UNCONN 0      0              127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=20))
udp   UNCONN 0      0           127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=18))
udp   UNCONN 0      0      151.80.146.40%ens3:68         0.0.0.0:*    users:(("systemd-network",pid=480,fd=10))
udp   UNCONN 0      0                 0.0.0.0:4789       0.0.0.0:*                                             
udp   UNCONN 0      0                 0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=11))
udp   UNCONN 0      0                    [::]:5355          [::]:*    users:(("systemd-resolve",pid=452,fd=13))
udp   UNCONN 0      0                       *:7946             *:*    users:(("dockerd",pid=18407,fd=28))      
tcp   LISTEN 0      128               0.0.0.0:49281      0.0.0.0:*    users:(("sshd",pid=14030,fd=3))          
tcp   LISTEN 0      4096           127.0.0.54:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=21))
tcp   LISTEN 0      4096        127.0.0.53%lo:53         0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=19))
tcp   LISTEN 0      4096              0.0.0.0:5355       0.0.0.0:*    users:(("systemd-resolve",pid=452,fd=12))
tcp   LISTEN 0      128                  [::]:49281         [::]:*    users:(("sshd",pid=14030,fd=4))          
tcp   LISTEN 0      4096                 [::]:5355          [::]:*    users:(("systemd-resolve",pid=452,fd=14))
tcp   LISTEN 0      4096                    *:7946             *:*    users:(("dockerd",pid=18407,fd=27))      

[7] CONFIG SSH
---------------------------------------------
Port 49281
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
UsePAM yes
X11Forwarding no
PrintMotd no
AcceptEnv LANG LC_*
Subsystem	sftp	/usr/lib/openssh/sftp-server
ClientAliveInterval 120
```

## pb/secure_servers.yml

```yaml
---
- name: Sécurisation des serveurs OVH Debian 12 (Mode Paranoïaque)
  hosts: swarm_nodes
  become: yes
  vars:
    sysadmin_user: "supadmin"
    sysadmin_password: "!" 
    new_ssh_port: 49281
    local_public_key: "~/.ssh/id_rsa.pub"
    
  tasks:
    # --- 1. CONFIGURATION SYSTEME & UTILISATEUR ---
    
    - name: Mise à jour du cache APT et upgrade
      apt:
        update_cache: yes
        upgrade: dist
        autoremove: yes

    - name: Création de l'utilisateur {{ sysadmin_user }}
      user:
        name: "{{ sysadmin_user }}"
        password: "{{ sysadmin_password }}"
        groups: sudo
        shell: /bin/bash
        append: yes
        state: present

    - name: Copie de la clé SSH pour {{ sysadmin_user }}
      authorized_key:
        user: "{{ sysadmin_user }}"
        state: present
        key: "{{ lookup('file', local_public_key) }}"

    - name: Sudo sans mot de passe pour {{ sysadmin_user }}
      copy:
        dest: "/etc/sudoers.d/{{ sysadmin_user }}"
        content: "{{ sysadmin_user }} ALL=(ALL) NOPASSWD: ALL"
        mode: '0440'
        validate: 'visudo -cf %s'

    - name: Création user dockremap (anticipation Docker)
      user:
        name: dockremap
        system: yes
        shell: /usr/sbin/nologin
        create_home: no
        state: present

    - name: Installation des paquets de sécurité
      apt:
        pkg:
          - nftables
          - fail2ban
          - sudo
          - curl
        state: present

    # --- 2. PARE-FEU (CRITIQUE) ---
    
    - name: Configuration NFTables (Double Porte)
      copy:
        dest: /etc/nftables.conf
        content: |
          #!/usr/sbin/nft -f
          flush ruleset

          table inet filter {
            chain input {
              type filter hook input priority 0; policy drop;
              iifname "lo" accept
              ct state established,related accept
              ip protocol icmp accept
              ip6 nexthdr icmpv6 accept

              # 1. On ouvre le NOUVEAU port
              tcp dport {{ new_ssh_port }} accept
              
              # 2. On garde l'ANCIEN port ouvert (Sécurité anti-coupure)
              tcp dport 22 accept
            }
            chain forward {
              type filter hook forward priority 0; policy drop;
            }
            chain output {
              type filter hook output priority 0; policy accept;
            }
          }
        mode: '0755'

    # SECURITE : On force le reload MAINTENANT, pas à la fin du playbook
    - name: Rechargement immédiat de NFTables
      service:
        name: nftables
        state: reloaded
        enabled: yes

    # --- 3. CONFIGURATION SSH (CRITIQUE) ---

    - name: Durcissement SSHD (Changement de port)
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: "{{ item.regexp }}"
        line: "{{ item.line }}"
      loop:
        - { regexp: '^#?Port', line: "Port {{ new_ssh_port }}" }
        - { regexp: '^#?PermitRootLogin', line: "PermitRootLogin no" }
        - { regexp: '^#?PasswordAuthentication', line: "PasswordAuthentication no" }
        - { regexp: '^#?PubkeyAuthentication', line: "PubkeyAuthentication yes" }
        - { regexp: '^#?X11Forwarding', line: "X11Forwarding no" }
        # On s'assure que SSH n'inclut pas d'autres fichiers qui écraseraient nos configs
        - { regexp: '^#?Include', line: "#Include /etc/ssh/sshd_config.d/*.conf" }

    # SECURITE : On vérifie la syntaxe avant de redémarrer
    - name: Vérification de la configuration SSHD
      command: sshd -t
      changed_when: false

    # On redémarre SSH. La session Ansible actuelle ne devrait pas couper,
    # car elle est "established" dans le pare-feu et SSH ne tue pas les processus fils actifs.
    - name: Redémarrage immédiat de SSH
      service:
        name: ssh
        state: restarted

    # --- 4. FAIL2BAN ---

    - name: Protection SSH avec Fail2Ban
      copy:
        dest: /etc/fail2ban/jail.d/ssh_custom.conf
        content: |
          [sshd]
          enabled = true
          port = {{ new_ssh_port }}
          bantime = 1h
          findtime = 10m
          maxretry = 5
      notify: Restart Fail2Ban

  handlers:
    - name: Restart Fail2Ban
      service: name=fail2ban state=restarted
```

## project_export.log

```text
[2025-12-04 22:56:15] Source  : .
[2025-12-04 22:56:15] Sortie  : project_export.md
[2025-12-04 22:56:15] Fichiers trouvés (avant filtre): 26
[2025-12-04 22:56:15] Fichiers à concaténer (après filtre): 25 (exclus auto:1 dir:0 file:0)
[2025-12-04 22:56:15] Concatène [1] deploy.yml (size=117)
[2025-12-04 22:56:15] Concatène [2] inventory.yml (size=887)
[2025-12-04 22:56:15] Concatène [3] pb/audit_security.yml (size=3561)
[2025-12-04 22:56:15] Concatène [4] pb/configure_firewall_swarm.yml (size=3897)
[2025-12-04 22:56:15] Concatène [5] pb/init_swarm.yml (size=1470)
[2025-12-04 22:56:15] Concatène [6] pb/reports/report_ovh.core.txt (size=10872)
[2025-12-04 22:56:15] Concatène [7] pb/reports/report_ovh.worker.01.txt (size=10677)
[2025-12-04 22:56:15] Concatène [8] pb/reports/report_ovh.worker.02.txt (size=10658)
[2025-12-04 22:56:15] Concatène [9] pb/secure_servers.yml (size=4293)

```

## roles/apps/tasks/main.yml

```yaml
---
# 1. Lecture Dynamique (Source de vérité)
- name: Lecture UID dockremap effectif
  shell: 'grep "^dockremap:" /etc/subuid | cut -d: -f2'
  register: remap_uid_out
  changed_when: false

- name: Lecture GID dockremap effectif
  shell: 'grep "^dockremap:" /etc/subgid | cut -d: -f2'
  register: remap_gid_out
  changed_when: false

- set_fact:
    remap_uid: "{{ remap_uid_out.stdout }}"
    remap_gid: "{{ remap_gid_out.stdout }}"

# 2. Réseau Overlay
- name: Réseau Traefik Public
  community.docker.docker_network:
    name: traefik-public
    driver: overlay
    attachable: true
    state: present

# -------------------------------------------------------
# GESTION DES DOSSIERS (SÉPARATION MANAGER / WORKERS)
# -------------------------------------------------------

# 1. DOSSIERS MANAGER UNIQUEMENT
# Ces services sont contraints sur "node.role == manager"
# Inutile de polluer les workers avec ces dossiers.
- name: Création dossiers Apps (Manager uniquement)
  file:
    path: "/srv/docker/{{ item }}"
    state: directory
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0755'
  loop:
    - traefik
    - traefik/acme
    - portainer
    - portainer/data
    - monitoring             # Parent
    - monitoring/prometheus
    - monitoring/grafana
    - monitoring/loki
  # Pas de delegate_to ici, ça s'exécute sur le manager (ovh.core)

# 2. DOSSIERS GLOBAUX (Promtail)
# Promtail tourne en mode "global" (sur tous les noeuds).
# Il a besoin de son dossier de config partout.
- name: Création dossiers Promtail (Cluster-wide)
  file:
    path: "/srv/docker/{{ item[1] }}"
    state: directory
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0755'
  # On ne boucle QUE sur les dossiers nécessaires à Promtail
  loop: "{{ groups['swarm_nodes'] | product(['monitoring', 'monitoring/promtail']) | list }}"
  delegate_to: "{{ item[0] }}"
  vars:
    ansible_become: yes

# 4. ACL Socket
- name: ACL Docker Socket
  acl:
    path: /var/run/docker.sock
    entity: "{{ remap_uid }}"
    etype: user
    permissions: rw
    state: present

- name: Login Docker Hub (Manager)
  community.docker.docker_login:
    username: "{{ docker_hub_user | default(omit) }}"
    password: "{{ docker_hub_pat | default(omit) }}"
  when: docker_hub_user is defined

# 5. Déploiement des Stacks

# --- TRAEFIK ---
- name: Configuration Traefik (Template)
  template:
    src: traefik.yaml.j2
    dest: /srv/docker/traefik/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Déploiement Stack Traefik
  community.docker.docker_stack:
    state: present
    name: traefik
    compose:
      - /srv/docker/traefik/docker-compose.yml
    prune: yes
    with_registry_auth: yes

# --- PORTAINER ---
- name: Configuration Portainer (Template)
  template:
    src: portainer.yaml.j2
    dest: /srv/docker/portainer/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Déploiement Stack Portainer
  community.docker.docker_stack:
    state: present
    name: portainer
    compose:
      - /srv/docker/portainer/docker-compose.yml
    prune: yes
    with_registry_auth: yes

# --- MONITORING ---
# A. Config Prometheus
- name: Configuration Prometheus (prometheus.yml)
  template:
    src: prometheus.yml.j2
    dest: /srv/docker/monitoring/prometheus/prometheus.yml
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0644'

# B. Config Promtail
- name: Configuration Promtail Cluster-wide (promtail.yaml)
  template:
    src: promtail.yaml.j2
    dest: /srv/docker/monitoring/promtail/config.yaml
    owner: "{{ remap_uid }}"
    group: "{{ remap_gid }}"
    mode: '0644'
  loop: "{{ groups['swarm_nodes'] }}"
  delegate_to: "{{ item }}"

# C. Compose Monitoring
- name: Configuration Monitoring (Template)
  template:
    src: monitoring.yaml.j2
    dest: /srv/docker/monitoring/docker-compose.yml
    owner: root
    group: root
    mode: '0644'

- name: Déploiement Stack Monitoring
  community.docker.docker_stack:
    state: present
    name: monitoring
    compose:
      - /srv/docker/monitoring/docker-compose.yml
    prune: yes
    with_registry_auth: yes
```

## roles/apps/templates/monitoring.yaml.j2

```text
version: '3.8'

services:
  # --- STOCKAGE DES LOGS ---
  loki:
    image: grafana/loki:3.0.0
    user: "0:0"
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

  # --- COLLECTEUR DE LOGS (Remplace le plugin Docker) ---
  promtail:
    image: grafana/promtail:2.9.0
    user: "0:0" # Root requis pour lire /var/lib/docker
    command: -config.file=/etc/promtail/config.yaml
    volumes:
      - /srv/docker/monitoring/promtail/config.yaml:/etc/promtail/config.yaml:ro
      - /var/lib/docker/{{ remap_uid }}.{{ remap_gid }}/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - monitoring
    deploy:
      mode: global # UN PAR NOEUD (Agent)
      placement:
        constraints: [node.platform.os == linux]

  # --- STOCKAGE DES MÉTRIQUES ---
  prometheus:
    image: prom/prometheus:v2.55.1
    user: "0:0"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=15d'
    volumes:
      # CORRECTION : On monte le FICHIER explicitement
      - /srv/docker/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      # On garde le dossier pour la data (si tu veux persister la TSDB, ajoute un volume ici)
      # - prometheus_data:/prometheus
    networks:
      - monitoring
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]

  # --- COLLECTEUR DE MÉTRIQUES SYSTÈME ---
  node-exporter:
    image: prom/node-exporter:v1.8.0
    user: "0:0" # Root souvent nécessaire pour les accès /proc /sys complets
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/host/root'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc|rootfs/var/lib/docker/containers|rootfs/var/lib/docker/overlay2|rootfs/run/docker/netns|rootfs/var/lib/docker/aufs)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/host/root:ro,rslave
    networks:
      - monitoring
    deploy:
      mode: global # UN PAR NOEUD
      placement:
        constraints: [node.platform.os == linux]

  # --- VISUALISATION ---
  grafana:
    image: grafana/grafana:11.0.0
    user: "0:0"
    volumes:
      - /srv/docker/monitoring/grafana:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://grafana.{{ main_domain }}
      - GF_SERVER_DOMAIN=grafana.{{ main_domain }}
      - GF_SERVER_SERVE_FROM_SUB_PATH=false
    networks:
      - monitoring
      - traefik-public
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.grafana.rule=Host(`grafana.{{ main_domain }}`)"
        - "traefik.http.routers.grafana.entrypoints=web,websecure"
        #- "traefik.http.routers.grafana.tls.certresolver=myresolver"
        - "traefik.http.services.grafana.loadbalancer.server.port=3000"

networks:
  monitoring:
    driver: overlay
    driver_opts:
      com.docker.network.driver.mtu: "1350"
  traefik-public:
    external: true
```

## roles/apps/templates/portainer.yaml.j2

```text
version: '3.8'

services:
  agent:
    # MISE A JOUR VERSION
    image: portainer/agent:2.21.5
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - agent_network
    deploy:
      mode: global
      placement:
        constraints: [node.platform.os == linux]

  portainer:
    # MISE A JOUR VERSION
    image: portainer/portainer-ce:2.21.5
    command: -H tcp://tasks.agent:9001 --tlsskipverify
    volumes:
      - /srv/docker/portainer/data:/data
    networks:
      - traefik-public
      - agent_network
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: [node.role == manager]
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.portainer.rule=Host(`portainer.{{ main_domain }}`)"
        - "traefik.http.routers.portainer.entrypoints=web"
        - "traefik.http.services.portainer.loadbalancer.server.port=9000"
        - "traefik.http.routers.portainer.entrypoints=websecure"
        - "traefik.http.routers.portainer.tls.certresolver=myresolver"

networks:
  traefik-public:
    external: true
  agent_network:
    driver: overlay
    attachable: true
    driver_opts:
      com.docker.network.driver.mtu: "1350"
```

## roles/apps/templates/prometheus.yml.j2

```text
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Auto-découverte des Node Exporters via le DNS Docker Swarm
  - job_name: 'node-exporter'
    dns_sd_configs:
      - names:
          - 'tasks.node-exporter'
        type: 'A'
        port: 9100
```

## roles/apps/templates/promtail.yaml.j2

```text
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    # C'est ICI que la magie opère : on utilise "docker_sd_configs"
    # Cela permet à Promtail de demander à Docker : "Qui sont ces conteneurs ?"
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: status
            values: [running]

    # "relabel_configs" sert à transformer les infos techniques de Docker
    # en labels lisibles pour toi dans Grafana/Loki.
    relabel_configs:
      # 1. Construction du chemin du log via l'ID du conteneur
      - source_labels: ['__meta_docker_container_id']
        target_label: '__path__'
        replacement: '/var/lib/docker/containers/$1/$1-json.log'

      # 2. Récupération du nom de la STACK (ex: monitoring, traefik)
      - source_labels: ['__meta_docker_container_label_com_docker_stack_namespace']
        target_label: 'stack'

      # 3. Récupération du nom du SERVICE (ex: monitoring_prometheus)
      - source_labels: ['__meta_docker_container_label_com_docker_swarm_service_name']
        target_label: 'service'

      # 4. Récupération du nom du CONTENEUR (ex: monitoring_prometheus.1.xB2...)
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        replacement: '$1'
        target_label: 'container'

      # 5. Récupération de l'ID du NODE (ex: ovh.worker.01)
      # Utile pour debugger un serveur spécifique
      - source_labels: ['__meta_docker_node_name']
        target_label: 'node_name'
```

## roles/apps/templates/traefik.yaml.j2

```text
version: '3.8'

services:
  traefik:
    image: traefik:v3.3
    command:
      # NOUVELLE SYNTAXE V3
      - --providers.swarm=true
      - --providers.swarm.endpoint=unix:///var/run/docker.sock
      - --providers.swarm.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --api.dashboard=true
      - --api.insecure=true
      - --accesslog=true
      - --metrics.prometheus=true
      - --metrics.prometheus.buckets=0.1,0.3,1.2,5.0
      - --accesslog=true
      - --log.level=INFO
      # Résolveur de certificats (à décommenter pour la prod)
      - --certificatesresolvers.myresolver.acme.email=admin@dgsynthex.online
      - --certificatesresolvers.myresolver.acme.storage=/acme/acme.json
      - --certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: host
      - target: 443
        published: 443
        protocol: tcp
        mode: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /srv/docker/traefik/acme:/acme
    networks:
      - traefik-public
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager
      labels:
        # On active Traefik pour lui-même
        - "traefik.enable=true"
        
        # On définit la règle de routage (Ton domaine)
        - "traefik.http.routers.dashboard.rule=Host(`traefik.{{ main_domain }}`)"
        
        # MAGIE : On pointe vers le service interne de l'API (pas besoin de port)
        - "traefik.http.routers.dashboard.service=api@internal"
        
        # On écoute sur le port 80 (web) et 443 (websecure)
        - "traefik.http.routers.dashboard.entrypoints=web,websecure"
        
        # (Optionnel) Authentification basique pour sécuriser l'IHM plus tard
        # - "traefik.http.routers.dashboard.middlewares=auth"
        - "traefik.http.services.dashboard.loadbalancer.server.port=8080"

networks:
  traefik-public:
    external: true
```

## roles/clean/tasks/main.yml

```yaml
---
# -------------------------------------------------------
# ETAPE : NETTOYAGE NUCLÉAIRE (AUTOMATISÉ)
# -------------------------------------------------------

- name: Arrêt des services Docker (Socket & Service)
  service:
    name: "{{ item }}"
    state: stopped
  loop:
    - docker.socket
    - docker
  ignore_errors: true

- name: Kill de sécurité des processus (shim, runc, containerd)
  shell: |
    pkill -9 dockerd || true
    pkill -9 containerd || true
    pkill -9 containerd-shim || true
    pkill -9 runc || true
  ignore_errors: true
  changed_when: false

# --- LA CORRECTION EST ICI ---
# On boucle sur /proc/mounts pour trouver TOUT ce qui est lié à Docker
# (Network namespaces, Overlay2, Containers, Volumes...)
# On trie à l'envers (-r) pour démonter les enfants avant les parents.
- name: Démontage FORCÉ de tous les points de montage Docker (Run & Lib)
  shell: |
    grep -E '/var/run/docker|/var/lib/docker' /proc/mounts | awk '{print $2}' | sort -r | xargs -r umount -f -l
  changed_when: false
  ignore_errors: true
  register: umount_result

- name: Debug du démontage
  debug:
    msg: "Nettoyage des montages terminé."
  when: umount_result.rc == 0

- name: Démontage SPÉCIFIQUE des Network Namespaces (Anti-Zombie)
  shell: |
    # 1. On démonte explicitement chaque fichier de namespace réseau
    # C'est souvent là que ça bloque (le fameux '1-zv77tswb8w')
    find /var/run/docker/netns -type f 2>/dev/null | xargs -r -I {} umount -l {}
    
    # 2. On démonte le dossier netns lui-même s'il est monté
    umount -l /var/run/docker/netns 2>/dev/null || true

    # 3. On finit par le balayage large classique
    grep -E '/var/run/docker|/var/lib/docker' /proc/mounts | awk '{print $2}' | sort -r | xargs -r umount -f -l
  changed_when: false
  ignore_errors: true
  args:
    executable: /bin/bash

# -----------------------------

- name: Destruction du stockage interne Docker (/var/lib/docker)
  file:
    path: /var/lib/docker
    state: absent

- name: Nettoyage des dossiers runtimes (/var/run/docker)
  file:
    path: /var/run/docker
    state: absent

- name: Suppression des données persistantes SaaS (/srv/docker)
  file:
    path: /srv/docker
    state: absent

# On redémarre Docker pour qu'il régénère ses dossiers proprement
- name: Redémarrage de Docker (Vierge)
  service:
    name: docker
    state: started

- name: Préparation dossier racine /srv/docker
  file:
    path: /srv/docker
    state: directory
    mode: '0755'
```

## roles/common/handlers/main.yml

```yaml
---
- name: Restart SSH
  service: name=ssh state=restarted

- name: Reload NFTables
  service: name=nftables state=reloaded
```

## roles/common/tasks/main.yml

```yaml
---
- name: Mise à jour du système
  apt:
    update_cache: true
    upgrade: dist
    autoremove: true

- name: Installation des paquets de base
  apt:
    pkg:
      - acl
      - curl
      - sudo
      - nftables
      - fail2ban
      - python3-pip
      - python3-yaml
      - python3-jsondiff
    state: present

# --- GESTION UTILISATEURS ---

- name: Verrouillage de l'utilisateur 'debian' (Sécurité)
  user:
    name: debian
    password_lock: true
    shell: /usr/sbin/nologin
    # On ne supprime pas pour éviter de casser cloud-init

- name: Création utilisateur Admin {{ sysadmin_user }}
  user:
    name: "{{ sysadmin_user }}"
    groups: sudo
    shell: /bin/bash
    append: yes
    state: present

- name: Clé SSH pour {{ sysadmin_user }}
  authorized_key:
    user: "{{ sysadmin_user }}"
    state: present
    # On ajoute un default pour éviter le crash si la variable manque
    key: "{{ lookup('file', local_public_key | default('~/.ssh/id_rsa.pub')) }}"

- name: Sudo sans mot de passe
  copy:
    dest: "/etc/sudoers.d/{{ sysadmin_user }}"
    content: "{{ sysadmin_user }} ALL=(ALL) NOPASSWD: ALL"
    mode: '0440'
    validate: 'visudo -cf %s'

# --- GESTION SUBUID / SUBGID (Séparation stricte) ---
# debian: 100000
# supadmin: 165536
# dockremap: 231072 (165536 + 65536)

- name: Configuration des SubUIDs
  copy:
    dest: /etc/subuid
    content: |
      debian:100000:65536
      {{ sysadmin_user }}:165536:65536
      dockremap:231072:65536
    mode: '0644'

- name: Configuration des SubGIDs
  copy:
    dest: /etc/subgid
    content: |
      debian:100000:65536
      {{ sysadmin_user }}:165536:65536
      dockremap:231072:65536
    mode: '0644'

# --- NFTABLES (Architecture Modulaire) ---

- name: Création du dossier de configuration modulaire NFTables
  file:
    path: /etc/nftables.d
    state: directory
    mode: '0755'

- name: Configuration Base NFTables (SSH uniquement)
  template:
    src: nftables_base.j2
    dest: /etc/nftables.conf
    mode: '0755'
    validate: '/usr/sbin/nft -c -f %s'
  notify: Reload NFTables

- name: Activation et Démarrage de NFTables
  service:
    name: nftables
    state: started
    enabled: yes

# --- CONFIGURATION SSHD ---

- name: Configuration SSHD
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
    validate: 'sshd -t -f %s' # Validation avant restart !
  loop:
    - { regexp: '^#?Port', line: "Port {{ ansible_port }}" }
    - { regexp: '^#?PermitRootLogin', line: "PermitRootLogin no" }
    - { regexp: '^#?PasswordAuthentication', line: "PasswordAuthentication no" }
  notify: Restart SSH

# --- FAIL2BAN ---

- name: Activation et Démarrage de Fail2Ban
  service:
    name: fail2ban
    state: started
    enabled: yes

# --- LOGGING NFTABLES ---

- name: Installation de Rsyslog
  apt:
    pkg: rsyslog
    state: present

- name: Configuration redirection logs NFTables
  copy:
    dest: /etc/rsyslog.d/10-nftables.conf
    content: |
      # Redirection des logs NFTables vers un fichier dédié
      :msg, contains, "[NFT-DROP]" -/var/log/nftables.log
      & stop
    mode: '0644'
  notify: Restart Rsyslog

- name: Configuration Logrotate pour NFTables (Évite de saturer le disque)
  copy:
    dest: /etc/logrotate.d/nftables
    content: |
      /var/log/nftables.log {
          rotate 7
          daily
          missingok
          notifempty
          delaycompress
          compress
          postrotate
              /usr/lib/rsyslog/rsyslog-rotate
          endscript
      }
    mode: '0644'
```

## roles/common/templates/nftables_base.j2

```text
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    
    # --- TRAFIC DE CONFIANCE (Loopback) ---
    iifname "lo" accept
    ct state established,related accept
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept

    # --- SSH ADMIN (Global) ---
    tcp dport {{ ansible_port }} accept
  }

  chain forward {
    type filter hook forward priority 0; policy drop;
    ct state established,related accept
  }

  chain output {
    type filter hook output priority 0; policy accept;
  }
}

# 1. On charge les règles spécifiques (Swarm, Apps...)
# Elles s'ajoutent à la suite des règles ci-dessus
include "/etc/nftables.d/*.conf"

# 2. LOGGING FINAL (Global)
# Cette règle est ajoutée TOUT À LA FIN de la chaîne input.
# Tout paquet qui n'a pas été accepté par le SSH ou par les includes (*) arrivera ici.
# On le logue dans /var/log/nftables.log (via rsyslog) avant qu'il soit jeté par la "policy drop".
add rule inet filter input limit rate 10/minute log prefix "[NFT-DROP] " flags all
```

## roles/docker/handlers/main.yml

```yaml
---
- name: Restart Docker
  service: name=docker state=restarted
```

## roles/docker/tasks/main.yml

```yaml
---
# --- 1. INSTALLATION & PRÉREQUIS ---

- name: Installation des dépendances système pour Docker
  apt:
    pkg:
      - ca-certificates
      - curl
      - gnupg
    state: present
    update_cache: true

- name: Création du dossier pour les clés GPG (si absent)
  file:
    path: /etc/apt/keyrings
    state: directory
    mode: '0755'

- name: Ajout de la clé GPG officielle Docker
  get_url:
    url: https://download.docker.com/linux/debian/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: '0644'
    force: true

- name: Ajout du dépôt Docker stable
  shell: |
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    tee /etc/apt/sources.list.d/docker.list > /dev/null
  args:
    creates: /etc/apt/sources.list.d/docker.list

- name: Mise à jour du cache APT après ajout du dépôt
  apt:
    update_cache: true

- name: Installation Docker Engine & Outils
  apt:
    pkg:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
      - python3-docker   # Requis pour les modules Ansible Docker
      - iptables         # Requis pour la compatibilité nftables
    state: present

# --- 2. CONFIGURATION UTILISATEUR & SÉCURITÉ ---

- name: Création utilisateur système dockremap
  user:
    name: dockremap
    system: yes
    state: present
    # Note : Les subuids sont gérés par le rôle 'common', on ne touche pas ici.

- name: Configuration Daemon Docker (UserNS + Logs + No Live Restore)
  copy:
    dest: /etc/docker/daemon.json
    content: |
      {
        "userns-remap": "default",
        "log-driver": "json-file",
        "log-opts": { "max-size": "10m", "max-file": "3" },
        "no-new-privileges": true,
        "min-api-version": "1.24"
      }
    mode: '0644'
  register: docker_config

# --- 3. GESTION INTELLIGENTE DU CHANGEMENT D'UID ---
# Si on change le subuid ou active le remap, il faut nettoyer /var/lib/docker
# sinon Docker plante au démarrage à cause des mauvaises permissions.

- name: Vérification existence dossier Docker
  stat:
    path: /var/lib/docker
  register: docker_dir

# - name: Nettoyage Docker si changement d'architecture UserNS
#   file:
#     path: /var/lib/docker
#     state: absent
#   # On supprime uniquement si la config daemon.json a changé ET que le dossier existait déjà
#   when: docker_config.changed and docker_dir.stat.exists
#   notify: Restart Docker

# --- 4. COMPATIBILITÉ RÉSEAU & SWARM ---

- name: Configuration alternatives iptables-nft (Fix Swarm Debian 12)
  alternatives:
    name: iptables
    path: /usr/sbin/iptables-nft
  notify: Restart Docker

# --- 5. DÉMARRAGE & FINALISATION ---

- name: Démarrage et Activation de Docker
  service:
    name: docker
    state: started
    enabled: yes

- name: Ajout de l'Admin au groupe Docker
  user:
    name: "{{ sysadmin_user }}"
    groups: docker
    append: yes

# --- 6. GESTION PERMISSIONS SOCKET (CRITIQUE POUR PORTAINER/TRAEFIK) ---

- name: Récupération de l'UID remappé (dockremap)
  shell: 'grep "^dockremap:" /etc/subuid | cut -d: -f2'
  register: dockremap_uid_check
  changed_when: false
  check_mode: no

- name: Application ACL sur le socket Docker pour le user remappé
  acl:
    path: /var/run/docker.sock
    entity: "{{ dockremap_uid_check.stdout }}"
    etype: user
    permissions: rw
    state: present
  when: dockremap_uid_check.stdout != ""
```

## roles/swarm/tasks/main.yml

```yaml
---
# -------------------------------------------------------
# ETAPE 1 : MANAGER
# -------------------------------------------------------

- name: Initialisation du Swarm (Manager)
  community.docker.docker_swarm:
    state: present
    advertise_addr: "{{ ansible_default_ipv4.address }}"
  register: swarm_init_result
  when: "'manager' in group_names"  # <--- CIBLAGE PAR GROUPE

- name: Récupération des Infos du Cluster (Token)
  community.docker.docker_swarm_info:
  register: swarm_facts
  delegate_to: "{{ groups['manager'][0] }}"
  run_once: true

# -------------------------------------------------------
# ETAPE 2 : PARE-FEU
# -------------------------------------------------------
# (Pas de changement ici, ça s'applique à tout le monde)
- name: Déploiement règles NFTables Swarm
  template:
    src: nftables_swarm.conf.j2
    dest: /etc/nftables.d/10-swarm.conf
    mode: '0644'
  notify: Reload NFTables

- name: Application immédiate du Pare-feu
  meta: flush_handlers

# -------------------------------------------------------
# ETAPE 3 : WORKERS - JONCTION ET NETTOYAGE
# -------------------------------------------------------

- name: Vérification état Swarm local du Worker
  community.docker.docker_host_info:
  register: worker_info
  when: "'workers' in group_names" # <--- CIBLAGE PAR GROUPE

- name: Quitter le mauvais Swarm
  command: docker swarm leave --force
  when:
    - "'workers' in group_names"
    - worker_info.host_info.Swarm.LocalNodeState == 'active'
    - worker_info.host_info.Swarm.Cluster.ID is defined
    - worker_info.host_info.Swarm.Cluster.ID != swarm_facts.swarm_facts.ID
  notify: Restart Docker

- name: Force restart Docker si nettoyage
  meta: flush_handlers

- name: Rejoindre le Swarm
  community.docker.docker_swarm:
    state: join
    join_token: "{{ swarm_facts.swarm_facts.JoinTokens.Worker }}"
    remote_addrs: [ "{{ hostvars[groups['manager'][0]]['ansible_default_ipv4']['address'] }}:2377" ]
  when:
    - "'workers' in group_names"
    - (worker_info.host_info.Swarm.LocalNodeState != 'active') or
      (worker_info.host_info.Swarm.Cluster.ID is defined and worker_info.host_info.Swarm.Cluster.ID != swarm_facts.swarm_facts.ID)

# -------------------------------------------------------
# ETAPE 6 : RÉSEAU GLOBAL (INFRASTRUCTURE)
# -------------------------------------------------------
- name: Création du réseau overlay 'traefik-public'
  community.docker.docker_network:
    name: traefik-public
    driver: overlay
    attachable: true # Important pour le debugging ou les conteneurs hors-stack
    driver_options:
      # encrypted: "yes"
      # FIX MTU CRITIQUE (OVH/OpenStack VXLAN)
      com.docker.network.driver.mtu: "1350"
  when: "'manager' in group_names"
  run_once: true
```

## roles/swarm/templates/nftables_swarm.conf.j2

```text
# Règles spécifiques Swarm
# Injectées dans la table "inet filter" définie dans nftables.conf

# Définition de la liste des IPs du cluster
define SWARM_NODES = { {{ groups['swarm_nodes'] | map('extract', hostvars, ['ansible_default_ipv4', 'address']) | join(', ') }} }

# Ajout à la chaîne INPUT existante
add rule inet filter input ip saddr $SWARM_NODES tcp dport { 2377, 7946 } accept
add rule inet filter input ip saddr $SWARM_NODES udp dport { 7946, 4789 } accept
add rule inet filter input ip protocol esp accept

# Ajout à la chaîne FORWARD existante (Pour les conteneurs et l'Overlay)
add rule inet filter forward iifname "docker0" accept
add rule inet filter forward oifname "docker0" accept
add rule inet filter forward iifname "docker_gwbridge" accept
add rule inet filter forward oifname "docker_gwbridge" accept

# Autoriser le trafic INTER-CONTENEURS sur les réseaux custom (Stacks)
# Docker nomme ces interfaces "br-<id_reseau>"
add rule inet filter forward iifname "br-*" accept
add rule inet filter forward oifname "br-*" accept

# Web Public (Uniquement sur le Manager)
{% if inventory_hostname == manager_host %}
add rule inet filter input tcp dport { 80, 443 } accept
{% endif %}
```

## site.yml

```yaml
---
- name: "Déploiement Infrastructure Swarm"
  hosts: swarm_nodes
  become: true
  
  # 1. On définit les handlers ici, au niveau global du Play
  handlers:
    - name: Restart Rsyslog
      service: name=rsyslog state=restarted

    - name: Reload NFTables
      service: name=nftables state=reloaded
      
    - name: Restart Docker
      service: name=docker state=restarted
      
    - name: Restart SSH
      service: name=ssh state=restarted

  # 2. On appelle les rôles
  roles:
    - role: clean
      tags: ["clean"]
    - role: common
      tags: ["common", "system"]
    - role: docker
      tags: ["docker", "engine"]
    - role: swarm
      tags: ["swarm", "network"]
    
- name: "Déploiement Applications"
  hosts: ovh.core
  become: true
  tags: ["apps", "deploy"]
  
  # On doit répéter les handlers ici car c'est un nouveau "Play"
  handlers:
    - name: Reload NFTables
      service: name=nftables state=reloaded
      
  roles:
    - apps
```

