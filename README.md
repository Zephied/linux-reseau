# linux-reseau

🧱 **Projet Réseau — Déploiement complet d'une architecture simple sous VirtualBox**

## 🎯 Objectif général

Mettre en place une infrastructure réseau complète composée de :

- 1 VM Firewall / Routeur
- 1 VM Web (HTTPS + éventuellement base de données)
- 1 VM Client jouant le rôle de NAS pour les sauvegardes
- 1 résolution DNS interne
- Accès SSH sécurisé
- Sauvegardes automatiques et centralisées

L’ensemble fonctionne dans VirtualBox en Host-Only + NAT, avec routage et accès Internet via la VM Firewall.

---

## 🖥️ 1. Création des machines virtuelles

### 1.1 Liste des VMs

| Machine | Rôle | Interfaces réseau |
|---------|------|-------------------|
| Firewall | Accès Internet, routage LAN, DNS | NAT (enp0s3), Host-Only (enp0s8) |
| Client / NAS | Stockage des sauvegardes (Samba) | Host-Only |
| Web | Serveur HTTP/HTTPS | Host-Only |

---

## 🌐 2. Configuration réseau VirtualBox

### 2.1 Interfaces utilisées

**NAT (accès Internet)**
- Seule la VM Firewall a un accès direct Internet.

**Host-Only**
- Tout le réseau interne est en `192.168.56.0/24`
- Toutes les VMs communiquent entre elles via cette interface.

**Schéma :**
```
Internet <-> NAT <-> FIREWALL <-> LAN Host-Only <-> {WEB, NAS}
```

---

## 🚧 3. Configuration du Firewall (routeur)

### 3.1 Donner une IP fixe à l'interface Host-Only

Éditer :
```bash
sudo nano /etc/network/interfaces
```

Ajouter :
```bash
auto enp0s8
iface enp0s8 inet static
   address 192.168.56.10
   netmask 255.255.255.0
```

### 3.2 Activer le routage
```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 3.3 Mettre en place le NAT (iptables)
```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo iptables -A FORWARD -i enp0s8 -j ACCEPT
sudo iptables -A FORWARD -i enp0s3 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

---

## 🔒 4. Configuration SSH sécurisée

**Sur toutes les machines :**

### 4.1 Installation OpenSSH serveur
```bash
sudo apt install ssh -y
```

### 4.2 Création d'une clé SSH (sur le PC hôte ou client)
```bash
ssh-keygen
```

### 4.3 Copie de la clé sur les serveurs
```bash
ssh-copy-id user@192.168.56.x
```

### 4.4 Désactivation du mot de passe
```bash
sudo nano /etc/ssh/sshd_config
```

Modifier :
```bash
PasswordAuthentication no
```

Redémarrer SSH :
```bash
sudo systemctl restart ssh
```

---

## 🌍 5. Serveur Web (VM WEB)

### 5.1 IP fixe
```bash
auto enp0s3
iface enp0s3 inet static
   address 192.168.56.30
   netmask 255.255.255.0
   gateway 192.168.56.10
   dns-nameservers 192.168.56.10
```

### 5.2 Installation Nginx
```bash
sudo apt install nginx -y
```

### 5.3 Certificat TLS auto-signé
```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
 -keyout /etc/ssl/private/nginx.key \
 -out /etc/ssl/certs/nginx.crt
```

Modifier la config nginx :
```bash
sudo nano /etc/nginx/sites-enabled/default
```

Activer HTTPS :
```nginx
listen 443 ssl;
ssl_certificate /etc/ssl/certs/nginx.crt;
ssl_certificate_key /etc/ssl/private/nginx.key;
```

Redémarrer :
```bash
sudo systemctl restart nginx
```

---

## 🧩 6. DNS interne (Bind9 installé sur le Firewall)

### 6.1 Installation
```bash
sudo apt install bind9 -y
```

6.2 Zone locale

Éditer :

```bash
sudo nano /etc/bind/named.conf.local
```


Ajouter :

```bash
zone "local.lan" {
    type master;
    file "/etc/bind/db.local.lan";
};
```

Créer la zone :

```bash
sudo nano /etc/bind/db.local.lan
```


Exemple :

```bash
$TTL 604800
@   IN  SOA ns.local.lan. admin.local.lan. (
    2 604800 86400 2419200 604800 )
@      IN  NS      ns.local.lan.
ns     IN  A       192.168.56.10
web    IN  A       192.168.56.30
nas    IN  A       192.168.56.20
```


Redémarrer :

```bash
sudo systemctl restart bind9
```


Test :

```bash
dig web.local.lan @192.168.56.10
```

📦 7. NAS Samba (VM Client)
7.1 Installation
```bash
sudo apt install samba -y
```

### 7.2 Dossier partagé
```bash
sudo mkdir -p /srv/sauvegardes
sudo chmod 2775 /srv/sauvegardes
sudo chown nobody:nogroup /srv/sauvegardes
```

### 7.3 Configuration Samba
```bash
sudo nano /etc/samba/smb.conf
```

Ajouter :
```ini
[sauvegardes]
   path = /srv/sauvegardes
   browseable = yes
   read only = no
   guest ok = yes
```

Redémarrer :
```bash
sudo systemctl restart smbd
```

---

## ♻️ 8. Sauvegardes automatiques

### 8.1 Montage du NAS (sur Web / Firewall)
```bash
sudo mkdir -p /mnt/sauvegardes
sudo mount -t cifs //192.168.56.20/sauvegardes /mnt/sauvegardes -o guest
```

### 8.2 Script de sauvegarde

Créer `/usr/local/bin/backup.sh` :
```bash
#!/bin/bash
DATE=$(date +"%Y-%m-%d_%H-%M")
DEST="/mnt/sauvegardes/$(hostname)"
mkdir -p "$DEST"

tar -czf "$DEST/backup_$DATE.tar.gz" /etc /var/www

find "$DEST" -type f -mtime +7 -delete
```

Rendre exécutable :
```bash
sudo chmod +x /usr/local/bin/backup.sh
```

### 8.3 Automatisation via cron
```bash
sudo crontab -e
```

Ajouter :
```cron
0 3 * * * /usr/local/bin/backup.sh >/var/log/backup.log 2>&1
```

---

## 📊 9. Monitoring avec Prometheus + Grafana

### 9.1 Prérequis

- VMs Debian 13 sur le même réseau local
- Une VM pour Prometheus + Grafana (ex : 192.168.56.10)
- Node Exporter installé sur toutes les VMs à surveiller

---

## 1️⃣ Installer Prometheus

### 1.1 Télécharger et installer Prometheus

Créer un utilisateur Prometheus :
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
```

Créer les dossiers nécessaires :
```bash
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

Télécharger la dernière version (exemple 2.45.0) :
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar xvf prometheus-2.45.0.linux-amd64.tar.gz
cd prometheus-2.45.0.linux-amd64
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus/
sudo chown -R prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool /etc/prometheus/consoles /etc/prometheus/console_libraries
```

### 1.2 Configurer Prometheus

Éditer `/etc/prometheus/prometheus.yml` :
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets:
          - '192.168.56.10:9100'
          - '192.168.56.20:9100'
          - '192.168.56.30:9100'
```

⚠️ Remplace les IP par celles de tes VMs
⚠️ Chaque VM doit avoir Node Exporter sur le port 9100

### 1.3 Créer le service systemd

Éditer `/etc/systemd/system/prometheus.service` :
```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
```

### 1.4 Démarrer Prometheus
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

Test :
```bash
curl http://192.168.56.10:9090/metrics
```

---

## 2️⃣ Installer Node Exporter sur toutes les VMs

Installer Node Exporter depuis le package Debian :
```bash
sudo apt update
sudo apt install -y prometheus-node-exporter
```

Démarrer le service :
```bash
sudo systemctl enable --now prometheus-node-exporter
sudo systemctl status prometheus-node-exporter
```

Tester l'accès :
```bash
curl http://<IP_VM>:9100/metrics
```

---

## 3️⃣ Installer Grafana

### 3.1 Télécharger et installer Grafana

Installer les dépendances :
```bash
sudo apt update
sudo apt install -y apt-transport-https software-properties-common wget
```

Ajouter la clé et le dépôt Grafana :
```bash
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

Installer Grafana :
```bash
sudo apt update
sudo apt install grafana
```

### 3.2 Démarrer Grafana
```bash
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server
```

Interface web : `http://192.168.56.10:3000`
Login par défaut : `admin / admin`

---

## 4️⃣ Ajouter Prometheus comme source de données dans Grafana

Menu ⚙️ → Data Sources → Add data source → Prometheus

URL :
```
http://192.168.56.10:9090
```

Clique sur « Save & Test » → doit afficher « Data source is working »

---

## 5️⃣ Importer le dashboard Node Exporter

Menu + → Import

Dashboard ID : `1860`

Sélectionne la source de données Prometheus

Clique sur Import

Tu verras alors toutes les métriques CPU, RAM, disque, réseau de tes VMs.

---

## 6️⃣ Vérifications finales

- ✅ Node Exporter actif sur toutes les VMs
- ✅ Prometheus scrape toutes les IP du fichier YAML
- ✅ Grafana récupère les métriques et le dashboard s'affiche

---

## 7️⃣ Optionnel : Alerting

- Grafana Alerting pour CPU, RAM, Disque
- Alertmanager pour emails, Slack ou Telegram

---

## 🐳 10. Installation de Docker et Docker Compose

### 10.1 Installer Docker et Docker Compose

Méthode simple depuis les dépôts Debian :
```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

Tester Docker :
```bash
docker --version
docker-compose --version
sudo docker run hello-world
```

✅ Si tout fonctionne, Docker est prêt à l'emploi.

---

### 10.2 Préparer l'arborescence du projet

```bash
sudo mkdir -p /srv/web-docker/{html,conf,certs}
cd /srv/web-docker
```

Structure :
- `html/` → Contient les fichiers du site web
- `conf/` → Contient les fichiers de configuration NGINX
- `certs/` → Contient les certificats SSL

---

### 10.3 Créer une page web de test

```bash
sudo nano html/index.html
```

Exemple de contenu :
```html
<h1>Serveur Web Docker OK</h1>
<p>Conteneur fonctionnel</p>
```

---

### 10.4 Configurer NGINX

Créer un fichier minimal `/srv/web-docker/conf/default.conf` :
```nginx
server {
    listen 80;
    server_name web.monlan.local;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

⚠️ Pour HTTPS, le serveur sera modifié ensuite pour inclure `listen 443 ssl;` et les certificats.

---

### 10.5 Créer le fichier Docker Compose

Fichier `/srv/web-docker/docker-compose.yml` :
```yaml
version: "3"

services:
  web:
    image: nginx:stable
    container_name: web_nginx
    ports:
      - "80:80"
      - "443:443"  # Ajouter si HTTPS
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./conf:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/ssl:ro  # Ajouter si HTTPS
    restart: always
```

---

### 10.6 Arrêter NGINX système (éviter les conflits)

```bash
sudo systemctl stop nginx
sudo systemctl disable nginx
sudo ss -tulnp | grep -E ':(80|443)'
```

⚠️ Assurez-vous qu'aucun service n'écoute sur les ports 80/443.

---

### 10.7 Lancer le serveur web Docker

```bash
cd /srv/web-docker
sudo docker-compose up -d
```

Vérifier le conteneur :
```bash
sudo docker ps
sudo docker logs web_nginx
```

Tester localement :
```bash
curl http://localhost
curl -k https://localhost  # si HTTPS activé
```

---

### 10.8 Accès depuis le réseau

Depuis une autre machine LAN :
```bash
curl http://192.168.56.30
curl -k https://web.monlan.local  # si HTTPS activé
```

- `192.168.56.30` → IP de la VM Web
- `web.monlan.local` → Nom configuré dans le DNS interne (Bind9)