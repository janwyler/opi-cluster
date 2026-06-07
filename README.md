# Orange Pi Cluster — Armbian + Docker Swarm

Ansible-Setup für einen Orange Pi Cluster mit Armbian Debian Trixie und Docker Swarm.

## Hardware

| Node     | Board           | Rolle    |
|----------|-----------------|----------|
| opiz2-1  | Orange Pi Zero 2W | Manager |
| opi3-2   | Orange Pi 3 LTS   | Worker  |
| opi3-3   | Orange Pi 3 LTS   | Worker  |

## Projektstruktur

```
opi-cluster/
├── ansible/
│   ├── ansible.cfg          # Ansible Konfiguration
│   ├── inventory.ini        # Node IPs und Gruppen
│   ├── hardening.yml        # Security Hardening
│   ├── docker_install.yml   # Docker auf alle Nodes
│   └── swarm_init.yml       # Docker Swarm initialisieren
├── stacks/
│   └── example-stack.yml    # Beispiel Docker Stack
└── docs/
    ├── Armbian_Cluster_Setup.pdf
    └── Docker_Swarm_Cluster.pdf
```

## Voraussetzungen

- Laptop mit Ubuntu / WSL
- Ansible installiert (`sudo apt install ansible -y`)
- SSH-Key vorhanden (`~/.ssh/id_ed25519`)
- Alle Nodes per SSH erreichbar

## Schnellstart

### 1. Inventory anpassen

```ini
# ansible/inventory.ini
[cluster]
opiz2-1 ansible_host=192.168.8.XXX
opi3-2  ansible_host=192.168.8.XXX
opi3-3  ansible_host=192.168.8.XXX
```

### 2. SSH-Keys kopieren (einmalig pro Node)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub janbo@192.168.8.XXX
```

### 3. Passwortloses sudo einrichten (einmalig pro Node)

```bash
ansible pi1 -m lineinfile -a \
  "dest=/etc/sudoers.d/janbo \
   line='janbo ALL=(ALL) NOPASSWD: ALL' \
   create=yes \
   mode=0440" \
  --become --ask-become-pass
```

### 4. Verbindung testen

```bash
cd ansible/
ansible cluster -m ping
```

### 5. Hardening ausführen

```bash
ansible-playbook hardening.yml
```

### 6. Docker installieren

```bash
ansible-playbook docker_install.yml
```

### 7. Docker Swarm initialisieren

```bash
ansible-playbook swarm_init.yml
```

## Nützliche Befehle

```bash
# Status aller Nodes
ansible cluster -a "uptime"

# Swarm Status
ansible manager -a "docker node ls"

# Service skalieren
ansible manager -a "docker service scale <name>=3"

# Stack deployen
ansible manager -m shell \
  -a "docker stack deploy -c /opt/stacks/example-stack.yml mystack"
```

## Dokumentation

Vollständige Anleitungen als PDF im `docs/` Verzeichnis:

- **Armbian_Cluster_Setup.pdf** — Flash, SSH, Ansible, Hardening
- **Docker_Swarm_Cluster.pdf** — Docker, Swarm, Services, Monitoring

## Erster SSH-Zugang (Armbian Erstlogin)

Nach dem ersten Boot per Ethernet verbinden:

```bash
# IP finden
nmap -sn 192.168.0.0/24

# Einloggen (Standard-Passwort: 1234)
ssh root@192.168.0.XXX
```

Armbian startet einen Wizard:
1. Neues Root-Passwort setzen
2. Neuen Benutzer anlegen (z.B. `YOUR_USERNAME`)
3. Passwort für neuen Benutzer setzen

Danach Hostname setzen:

```bash
sudo hostnamectl set-hostname pi1
sudo sed -i 's/127.0.1.1.*/127.0.1.1\tpi1/' /etc/hosts
sudo reboot now
```

Für jeden Pi wiederholen: `pi1`, `pi2`, `pi3` usw.
