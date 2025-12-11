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