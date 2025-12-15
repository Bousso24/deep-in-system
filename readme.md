# DEEP-IN-NET
Mettre en place un serveur Linux sécurisé et fonctionnel hébergeant un site WordPress, tout en appliquant les bonnes pratiques de sécurité, de gestion des utilisateurs, de services réseau et de sauvegarde.
## La partie machine virtuelle :
- Installez la dernière version LTS d'un serveur Ubuntu en tant que machine virtuelle.
- La taille du disque de la machine virtuelle doit être de 30 Go.
- Vous devez diviser le disque de votre machine virtuelle en partitions comme suit : swap:4 Go / : 15 Go /home : 5 Go /backup : 6 Go
- Votre nom d'utilisateur doit être votre nom de connexion.
- Vous devez définir votre nom d'hôte au format suivant : {username}-host, si votre identifiant est potato, alors votre nom d'hôte doit être potato-host.
1. SWAP : 4 Go → Mémoire d'urgence
Mémoire virtuelle quand la RAM est pleine
Avantages :
Évite le plantage quand trop de programmes ouverts
Permet l'hibernation (sauvegarde de l'état sur disque)
Sans swap : Le système plante si la RAM est saturée
2. / (racine) : 15 Go → Système d'exploitation
Contient : Ubuntu, noyau, programmes système
Pourquoi 15 Go ? Ubuntu + programmes système ≈ 10-12 Go
Avantages :
Isoler le système des données utilisateurs
Réinstallation sans toucher aux données
Limiter l'impact si / est plein
3. /home : 5 Go → Données utilisateurs
Contient : Fichiers des utilisateurs (luffy, zoro, nami)
Pourquoi séparé ? Pour réinstaller Ubuntu sans perdre vos fichiers
Avantages :
Sauvegarde facile (juste /home)
Migration vers une nouvelle installation
Quotas par utilisateur possible
Sécurité : Empêche un utilisateur de remplir /
4. /backup : 6 Go → Sauvegardes système
Contenu : Sauvegardes WordPress (ta mission)
Pourquoi séparé ? Protection maximale
Avantages :
Même si / est corrompu, les backups sont saufs
Peut être monté en "read-only" pour sécurité
N'affecte pas les autres partitions si plein
Important : C'est POUR TON PROJET (backup WordPress)
## La partie réseau :
- Configurez une adresse IP privée statique 
Netplan est le gestionnaire reseau par defaut(statique ou DHCP). Pour fixer une adresse IP statique persistante,
 il faut configurer Netplan dans /etc/netplant/50-cloud-init.yaml et le modifier comme suit:
 ```bash
network:
  version: 2
  ethernet:
    enp0s3:
      dhcp4: no
      adresses:
      - 192.168.1.100/24
      routes:
      - to: default
        via: 192.168.1.1
      nameservers:
        adresses:
        - 8.8.8.8
        - 1.1.1.1
 ```
 Vous devez pouvoir vous connecter à Internet ! Vous pouvez tester avec :
 $> ping -c 5 google.com
 
 ## La partie relative à la sécurité Secure SHell (SSH):
 Un serveur SSH sert à :
- Se connecter à distance
- De manière sécurisée (connexion chiffrée)
- Avec une authentification fiable (mot de passe ou clé publique)
- Pour gérer, administrer ou transférer des fichiers
- Ou créer des tunnels sécurisés
C’est un outil indispensable pour tout administrateur système.

dans le fichier sudo nano /etc/ssh/sshd_config, modifions le port en 2222 désactiver la connexion root à distance via ssh.
```bash
Port 2222
PermitRootLogin no
```
🚫 2. Pourquoi désactiver le login root via SSH ?
Le fichier :
/etc/ssh/sshd_config
doit contenir :
```bash
PermitRootLogin no
```
Cela empêche les hackers (ou un auditeur) de :
- se connecter directement en root
- faire un brute-force sur root
- contourner la sécurité de sudo

Puis redémarrer :
sudo systemctl restart ssh

Vérifie que ton serveur écoute bien sur 2222:
sudo ss -tlnp | grep ssh

*Configurez le pare-feu 
Fermer tout par défaut :
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
Ouvrir SSH sur 2222 :
```bash
sudo ufw allow 2222/tcp
sudo ufw reload
```
Activer le firewall :
```bash
sudo ufw enable
```
Vérifier :
```bash
sudo ufw status numbered
```
Remarque:
Si tu fais tous ces changements et rien ne change:
Parce que systemd utilise ssh.socket pour lancer SSH automatiquement.
- ta configuration n’est pas prise en compte
- SSHD démarre à partir du socket systemd, pas à partir du service classique
- Et ce socket force SSH à écouter sur le port 22, même si ton fichier sshd_config dit 2222.
C’est un piège classique sur Ubuntu Server récent.
Ce socket est défini ici :
```bash
/usr/lib/systemd/system/ssh.socket
```
Et il contient :
```bash
ListenStream=22
```
➡️ Donc systemd force l’écoute sur le port 22, et ignore ton changement.
✅ SOLUTION : Désactiver ssh.socket et activer ssh.service
On va faire en sorte que ce soit le service SSH qui contrôle le port, pas le socket.
1️⃣ Désactiver le socket
```bash
sudo systemctl stop ssh.socket
sudo systemctl disable ssh.socket
```
2️⃣ Activer et redémarrer le service SSH classique
```bash
sudo systemctl enable ssh.service
sudo systemctl restart ssh.service
```
3️⃣ Vérifier
```bash
sudo ss -tlnp | grep ssh
```
🎉 Après ça, teste la connexion depuis ta machine hôte
```bash
ssh -p 2222 bousso-host@192.168.1.150
```
 ## Partie Gestion des utilisateurs :
 Nous allons cree deux utlisateurs l un s authentifie avec une cle et l autre avec mot de passe.
 Sur notre machine hote nous allons generer une paire de cle SSH avec cette commande:

 ### Sur votre ordinateur local
```bash
ssh-keygen -t rsa -b 4096 -C "luffy@server" -f ~/.ssh/id_luffy
```
- Résultat :
→ ~/.ssh/id_luffy      → Clé PRIVÉE (gardez-la !)
→ ~/.ssh/id_luffy.pub  → Clé PUBLIQUE (à copier sur le serveur)

 * Pour creer l utilsateur luffy tapons cette commande:
 ```bash
 - sudo adduser --home /home/luffy --disabled-password luffy
 ```
→ Crée un utilisateur luffy qui ne pourra pas se connecter avec un mot de passe et definir /home/luffy comme repertoire personnel.
 - ajouter luffy au groupe sudo avec la commande:
 ```bash
 sudo usermod -aG sudo luffy 
 ```
 avec:
 . usermod : Modifie un utilisateur existant
 . -aG : Ajoute aux Groupes (sans écraser les groupes existants)
 . sudo : Groupe qui donne les privilèges administrateur
→ luffy peut utiliser sudo pour exécuter des commandes admin
 - Créer le dossier .ssh où seront stockées les clés SSH
 ```bash
 sudo mkdir -p /home/luffy/.ssh
 ```
 . mkdir : Crée un répertoire
 . -p : Crée les répertoires parents si nécessaire
 . /home/luffy/.ssh : Dossier où seront stockées les clés SSH
→ C'est un dossier caché (commence par .)
 - Changer le propriétaire:
 ```bash
 sudo chown -R luffy:luffy /home/luffy/.ssh 
 ```
 . chown : Change le propriétaire
 . -R : Récursif (dossier et contenu)
 . luffy:luffy : Propriétaire:groupe
→L'utilisateur luffy devient propriétaire de son dossier .ssh
dans cette dossier on va cree un fichier /home/luffy/.ssh/authorized_keys qui vas stocker la cle publique

- Sécurisons nos dossiers:
→ sudo chmod 700 /home/luffy/.ssh : seul luffy peut acceder a son dossier .ssh
→ sudo chmod 600 /home/luffy/.ssh/authorized_keys: Empêche la modification par d'autres utilisateurs

* Pour creer l utilsateur luffy tapons cette commande:
```bash
sudo adduser --home /home/zoro /zoro
```
Cree un utlisateur zoro qui se connecte avec un mot de passe et definir /home/zoro comme repertoire personnel
. Configuration du serveur SSH:
dans le fichier sudo nano /etc/ssh/sshd_config , nous allons ajouter des modifications pour l autorisation des connexions
```bash
Match User luffy
    PasswordAuthentication no
    PubkeyAuthentication yes

Match User zoro
    PasswordAuthentication yes
    PubkeyAuthentication no```
```    
* Redémarrer le service SSH:
```bash
sudo systemctl restart ssh
```
## Partie Services :
1. mettre à jour les paquets :
```bash
sudo apt update && sudo apt upgrade -y
```
Puis installer le server ftp avec vsftpd
```bash
sudo apt install vsftpd -y
```
2. Creons notre utilisateur nami sans shell de connexion
```bash
sudo adduser --shell /bin/false --home /home/nami nami
```
- --shell /bin/false : Empêche la connexion SSH/shell
- --home /home/nami : Répertoire personnel

3. Donner à nami un accès uniquement sur /backup
Créer le répertoire backup:
sudo mkdir /backup
Donner /backup comme “home directory” pour nami
```bash
sudo usermod -d /backup nami
```
4. Mettre /backup en lecture seule pour nami
```bash
sudo chown root:root /backup
sudo chmod 755 /backup
```
root:root → seul root peut écrire dans /backup
755 → tout le monde peut lire / exécuter → donc FTP lecture seule

5. Configurer vsftpd
```bash
sudo nano /etc/vsftpd.conf
```
Vérifie/modifie les lignes suivantes :
```bash
anonymous_enable=NO
local_enable=YES
write_enable=NO
chroot_local_user=YES
allow_writeable_chroot=YES
```
Explications :
- anonymous_enable=NO → interdit les connexions anonymes
- local_enable=YES → autorise les comptes du système (comme nami)
- write_enable=NO → empêche tout upload → lecture seule
- chroot_local_user=YES → enferme l’utilisateur dans son dossier (jail)
- allow_writeable_chroot=YES → évite une erreur si jamais

6. Redémarrer vsftpd
```bash
sudo systemctl restart vsftpd
```
## La partie base de données :
MySQL est un système de gestion de bases de données relationnelles (SGBDR) open-source.
En français simple : C'est un logiciel qui stocke, organise et gère des données de manière structurée.

1. Installation de MySQL Server
```bash
sudo apt update
sudo apt install mysql-server -y
```
2. Sécurisation de MySQL
Exécuter le script de sécurisation
```bash
sudo mysql_secure_installation
```
Réponses recommandées :

. VALIDATE PASSWORD COMPONENT? → Y
. New password for root: → [Choisir un mot de passe FORT]
. Remove anonymous users? → Y
. Disallow root login remotely? → Y  (IMPORTANT!)
. Remove test database? → Y
. Reload privilege tables? → Y

3. Vérifier la configuration de sécurité
Vérifions que root ne peut se connecter qu’en local
```bash
sudo mysql
SELECT user, host FROM mysql.user;
```
Tu dois voir :
```bash
root | localhost
```
4.  Désactiver les connexions externes
Bloquer MySQL à l'intérieur du serveur uniquement
Édite le fichier de configuration :
```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
Vérifie cette ligne :
```bash
bind-address = 127.0.0.1
```
Cela signifie :

👉 MySQL n’écoute que sur localhost (connexion interne).
👉 Il est invisible depuis l’extérieur → ultra sécurisé.

Redémarre MySQL :
```bash
sudo systemctl restart mysql
```
ne jamais ouvrir le port 3306 !
MySQL n’est pas un service destiné à être accessible depuis Internet.
Donc personne dans le réseau n’a besoin d’y accéder.
Ouvrir 3306 est inutile → donc dangereux.
Ton ufw doit bloquer 3306 :
```bash
sudo ufw deny 3306
```
5. Créer la base WordPress
Se connecter à MySQL
```bash
sudo mysql -u root -p
```
Créer la base de données WordPress
```bash
CREATE DATABASE wordpress_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```
Créer l'utilisateur dédié
--> Créer l'utilisateur (localhost seulement)
```bash
CREATE USER 'wp_user'@'localhost' IDENTIFIED BY 'VotreMotDePasseFort123!';
```
--> Donner tous les droits sur la base WordPress
```bash
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wp_user'@'localhost';
```
--> Appliquer les changements
```bash
FLUSH PRIVILEGES;
```
Vérifier les permissions
```bash
SHOW GRANTS FOR 'wp_user'@'localhost';
```
6. Vérifications de sécurité
Vérifier que root ne peut pas se connecter à distance
```bash
sudo mysql -u root -p -e "SELECT user, host FROM mysql.user WHERE user='root';"
```
## Partie WordPress : Installation et Configuration
🎯 Objectifs :
Installer WordPress dans la racine web
Configurer pour qu'il fonctionne normalement
Sécuriser wp-config.php
Tester avec la création de contenu

1. Prérequis
Vérifier que LAMP est installé:
--> Apache
```bash
sudo systemctl status apache2
```
--> MySQL (déjà fait)
```bash
sudo systemctl status mysql
```
--> PHP
```bash
php --version
```
2. Télécharger WordPress
Se placer dans la racine web
```bash
cd /var/www/html/
```
--> Télécharger la dernière version
```bash
sudo wget https://wordpress.org/latest.tar.gz
```
--> Extraire
```bash
sudo tar -xzvf latest.tar.gz
```
--> Déplacer les fichiers à la racine
```bash
sudo mv wordpress/* .
sudo rmdir wordpress/
```
--> Nettoyer
```bash
sudo rm latest.tar.gz
```
3. Configurer les permissions
Définir le propriétaire correct
--> Définir www-data comme propriétaire (Apache)
```bash
sudo chown -R www-data:www-data /var/www/html/
```
--> Permissions sécurisées
```bash
sudo find /var/www/html/ -type d -exec chmod 755 {} \;
sudo find /var/www/html/ -type f -exec chmod 644 {} \;
```
--> Vérifier
```bash
ls -la /var/www/html/
```
4. Configuration WordPress
Créer wp-config.php
```bash
cd /var/www/html/
```
--> Copier le fichier de configuration exemple
```bash
sudo cp wp-config-sample.php wp-config.php
```
--> Éditer avec les infos MySQL
```bash
sudo nano wp-config.php
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wp_user');
define('DB_PASSWORD', 'TON_MOT_DE_PASSE');
define('DB_HOST', 'localhost');
```
5. Finaliser via le navigateur
```bash
http://192.168.1.150
```
## Backup : Configuration des sauvegardes
Les sauvegardes protègent contre les erreurs humaines, les pannes matérielles, les attaques de virus, les coupures de courant et les catastrophes naturelles. Elles permettent de gagner du temps et de l'argent en cas de survenance de tels incidents.

Dans cet exercice, vous allez mettre en place une méthode de sauvegarde simple à l'aide de tâches cron.
Cette partie permet de protéger, automatiser et sécuriser les données d’un site web en production.
Créer un système automatique qui fait une copie de sécurité de ton site WordPress tous les jours à minuit.

1. Créer le script de backup
```bash
sudo nano /usr/local/bin/wp_backup.sh
```
Contenant du script
```bash
#!/bin/bash

# Variables
DATE=$(date +"%Y-%m-%d_%H-%M-%S")
BACKUP_DIR="/backup"
LOG_FILE="/var/log/backup.log"

DB_NAME="wordpress_db"
DB_USER="wp_user"
DB_PASS="VotreMotDePasseFort123!"

BACKUP_FILE="$BACKUP_DIR/wp_db_backup_$DATE.sql"
TAR_FILE="$BACKUP_DIR/wp_db_backup_$DATE.tar.gz"

# Dump de la base
mysqldump -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" > "$BACKUP_FILE"

if [ $? -eq 0 ]; then
    tar -czf "$TAR_FILE" -C "$BACKUP_DIR" "$(basename "$BACKUP_FILE")"
    rm "$BACKUP_FILE"
    echo "$DATE - Backup WordPress réussi" >> "$LOG_FILE"
else
    echo "$DATE - ERREUR backup WordPress" >> "$LOG_FILE"
fi
```
Le fichier /var/log/backup.log permet :
de prouver que le backup a bien eu lieu
de savoir quand
de détecter les erreurs

Donner les permissions correctes
```bash
sudo chmod 700 /usr/local/bin/wp_backup.sh
sudo chown root:root /usr/local/bin/wp_backup.sh
```
2. Créer le cron job (00:00)
```bash
sudo crontab -e
0 0 * * * /usr/local/bin/wp_backup.sh
```
Explication:
Sauvegarde tous les jours à 00:00

3. Vérifier /backup pour FTP (nami)
```bash
sudo chown root:root /backup
sudo chmod 755 /backup
```
nami :
✅ peut lire
❌ ne peut pas écrire
❌ ne peut pas supprimer

Test manuel:
```bash
ls -lh /backup
wp_db_backup_2025-XX-XX_00-00-00.tar.gz
```
Et le log :
```bash
cat /var/log/backup.log
```
Exemple attendu :
```bash
2025-12-13_00-00-01 - Backup WordPress réussi
```


