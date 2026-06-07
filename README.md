# Orange Pi Cluster — Armbian + Docker Swarm

Ansible-Setup für einen Orange Pi Cluster mit Armbian Debian Trixie und Docker Swarm.

## Hardware

| Node     | Board             | Rolle   |
|----------|-------------------|---------|
| pi1      | Orange Pi Zero 2W | Manager |
| pi2      | Orange Pi 3 LTS   | Worker  |
| pi3      | Orange Pi 3 LTS   | Worker  |

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
└── docs/                    # Anleitungen (lokal, nicht in git)
```

## Setup — Schritt für Schritt

### 1. Erster SSH-Zugang (Armbian Erstlogin)

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

### 2. Hostname setzen (pro Pi)

```bash
sudo hostnamectl set-hostname pi1
sudo sed -i 's/127.0.1.1.*/127.0.1.1\tpi1/' /etc/hosts
```

> Für jeden Pi wiederholen: `pi1`, `pi2`, `pi3` usw.

### 3. System aktualisieren

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot now
```

### 4. SSH-Key erstellen (einmalig auf Laptop)

```bash
# Prüfen ob bereits vorhanden
ls ~/.ssh/id_ed25519

# Falls nicht — erstellen
ssh-keygen -t ed25519 -C "ansible-cluster"
# Passphrase leer lassen → Enter Enter
```

### 5. SSH-Keys kopieren (einmalig pro Node, interaktiv)

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub YOUR_USERNAME@192.168.0.XXX  # pi1
ssh-copy-id -i ~/.ssh/id_ed25519.pub YOUR_USERNAME@192.168.0.XXX  # pi2
ssh-copy-id -i ~/.ssh/id_ed25519.pub YOUR_USERNAME@192.168.0.XXX  # pi3
```

### 6. Ansible installieren (Laptop)

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

### 7. Inventory anpassen

```bash
nano ansible/inventory.ini
# IPs und YOUR_USERNAME eintragen
```

### 8. Passwortloses sudo einrichten (einmalig pro Node)

```bash
# Jeden Pi einzeln mit seinem sudo-Passwort
ansible pi1 -m lineinfile -a \
  "dest=/etc/sudoers.d/YOUR_USERNAME \
   line='YOUR_USERNAME ALL=(ALL) NOPASSWD: ALL' \
   create=yes \
   mode=0440" \
  --become --ask-become-pass

# Wiederholen für pi2, pi3 usw.
```

> Danach funktioniert `--become` ohne Passwort-Abfrage auf allen Nodes.

### 9. Verbindung testen

```bash
cd ansible/
ansible cluster -m ping
ansible cluster -m command -a 'whoami' --become
```

### 10. Hardening ausführen

```bash
ansible-playbook hardening.yml
```

> Beim Ausführen wird gefragt ob Passwort-Login deaktiviert werden soll (Standard: no).
> Erst `yes` eingeben wenn SSH-Key-Login funktioniert!

### 11. Docker installieren

```bash
ansible-playbook docker_install.yml
```

### 12. Docker Swarm initialisieren

```bash
ansible-playbook swarm_init.yml
```

> Manager wird automatisch initialisiert, Worker joinen automatisch.

### Troubleshooting: Worker im falschen Zustand

```bash
# Worker aus altem Swarm-Versuch befreien
ansible workers -a "docker swarm leave --force" --become

# Nochmals joinen
ansible-playbook swarm_init.yml
```

## Nützliche Befehle

```bash
# Status aller Nodes
ansible cluster -a "uptime"

# Swarm Status
ansible manager -a "docker node ls" --become

# Alle Nodes updaten
ansible cluster -m apt -a "update_cache=yes upgrade=full" --become

# Alle neu starten
ansible cluster -a "reboot" --become

# Stack deployen
ansible manager -m shell \
  -a "docker stack deploy -c /opt/stacks/example-stack.yml mystack" --become
```
