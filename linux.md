# TP Avancé : "Mission Ultime : Sauvegarde et Sécurisation"
## Étape 1 : Analyse et nettoyage du serveur
### Lister les tâches cron pour détecter des backdoors :
Analysez les tâches cron de tous les utilisateurs pour identifier celles qui semblent malveillantes.
````powershell
[root@localhost ~]# for user in $(cut -f1 -d: /etc/passwd); do crontab -u $user -l; done
````
### Identifier et supprimer les fichiers cachés :
Recherchez les fichiers cachés dans les répertoires /tmp, /var/tmp et /home.
````powershell
[root@localhost ~]# cd /tmp
[root@localhost tmp]# ls -a
...
[root@localhost tmp]# cd /var/tmp
[root@localhost tmp]# ls -a
...
[root@localhost tmp]# cd /home
[root@localhost home]# ls -a
...
````
Supprimez tout fichier suspect ou inconnu.
````powershell
[root@localhost home]# rm -f /tmp/malicious.sh
[root@localhost home]# rm -f /var/tmp/.nop
[root@localhost home]# rm -rf /home/attacker
````
### Analyser les connexions réseau actives :
Listez les connexions actives pour repérer d'éventuelles communications malveillantes.
````powershell
[root@localhost home]# netstat -tuln
````
## Étape 2 : Configuration avancée de LVM
### Créer un snapshot de sécurité pour /mnt/secure_data :
Prenez un snapshot du volume logique secure_data.
````powershell
[root@localhost secure_data]# cd /mnt/secure_data
[root@localhost secure_data]# lvcreate --size 100M --snapshot --name secure_data_snapshot /dev/vg_secure/secure_data
````
### Tester la restauration du snapshot :
Supprimez un fichier dans /mnt/secure_data.
````powershell
[root@localhost secure_data]# ls -a
.  ..  lost+found  sensitive1.txt  sensitive2.txt
[root@localhost secure_data]# rm sensitive1.txt
[root@localhost secure_data]# ls -a
.  ..  lost+found  sensitive2.txt
````
Montez le snapshot et restaurez le fichier supprimé.
````powershell
[root@localhost secure_data]# mkdir /mnt/snapshot_mount
[root@localhost secure_data]# mount /dev/vg_secure/secure_data_snapshot /mnt/snapshot_mount
[root@localhost secure_data]# cp /mnt/snapshot_mount/sensitive1.txt /mnt/secure_data/
[root@localhost secure_data]# umount /mnt/snapshot_mount
[root@localhost secure_data]# rmdir /mnt/snapshot_mount
[root@localhost secure_data]# ls -a
.  ..  lost+found  sensitive1.txt  sensitive2.txt
````
### Optimiser l’espace disque :
Si le volume logique secure_data est plein, étendez-le en ajoutant de l’espace à partir du groupe de volumes existant.
````powershell
[root@localhost ~]# lvextend -L +2G /dev/mapper/vg_secure-snap
````
## Étape 3 : Automatisation avec un script de sauvegarde
### Créer un script secure_backup.sh :
Archive le contenu de /mnt/secure_data dans /backup/secure_data_YYYYMMDD.tar.gz.
Exclut les fichiers temporaires (.tmp, .log) et les fichiers cachés.
### Ajoutez une fonction de rotation des sauvegardes :
Conservez uniquement les 7 dernières sauvegardes pour économiser de l’espace.
Archive le contenu de /mnt/secure_data dans /backup/secure_data_YYYYMMDD.tar.gz.
````powershell
[root@localhost ~]# nano /usr/local/bin/secure_backup.sh
````
````bash
#!/bin/bash

# Variables
SOURCE_DIR="/mnt/secure_data"
BACKUP_DIR="/backup"
DATE=$(date +\%Y\%m\%d)
ARCHIVE_NAME="secure_data_${DATE}.tar.gz"
EXCLUDE_LIST="--exclude='*.tmp' --exclude='*.log' --exclude='.*'"

# Créer l'archive
tar -czvf $BACKUP_DIR/$ARCHIVE_NAME $EXCLUDE_LIST $SOURCE_DIR

# Supprimer les sauvegardes anciennes (plus de 7 jours)
find $BACKUP_DIR -name "secure_data_*.tar.gz" -type f -mtime +7 -exec rm -f {} \;
````
````powershell
[root@localhost ~]# chmod +x /usr/local/bin/secure_backup.sh
````
### Testez le script :
Exécutez le script manuellement et vérifiez que les archives sont créées correctement.
````powershell
[root@localhost ~]# /usr/local/bin/secure_backup.sh
````
### Automatisez avec une tâche cron :
Planifiez le script pour qu’il s’exécute tous les jours à 3h du matin.
````powershell
[root@localhost ~]# crontab -e
````
````bash
...
0 3 * * * /usr/local/bin/secure_backup.sh
...
````
## Étape 4 : Surveillance avancée avec auditd
### Configurer auditd pour surveiller /etc :
Ajoutez une règle avec auditctl pour surveiller toutes les modifications dans /etc.
````powershell
[root@localhost ~]# auditctl -w /etc -p wa -k etc_changes
````powershell
touch /etc/profile
````
### Analyser les événements :
Recherchez les événements associés à la règle configurée et exportez les logs filtrés dans /var/log/audit_etc.log.
````powershell
ausearch -k etc_changes
ausearch -k etc_changes > /var/log/audit_etc.log
````
## Étape 5 : Sécurisation avec Firewalld
### Configurer un pare-feu pour SSH et HTTP/HTTPS uniquement :
Autorisez uniquement les ports nécessaires pour SSH et HTTP/HTTPS.
````powershell
[root@localhost ~]# firewall-cmd --zone=public --add-port=22/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=80/tcp --permanent
[root@localhost ~]# firewall-cmd --zone=public --add-port=443/tcp --permanent
````
Bloquez toutes les autres connexions.
````powershell
[root@localhost ~]# firewall-cmd --zone=public --set-target=DROP --permanent
[root@localhost ~]# firewall-cmd --reload
````
## Bloquer des IP suspectes :
À l’aide des logs d’audit et des connexions réseau, bloquez les adresses IP malveillantes identifiées.
````powershell
[root@localhost ~]# firewall-cmd --zone=public --add-rich-rule="rule family='ipv4' source address='192.168.1.100' drop" --permanent
[root@localhost ~]# firewall-cmd --reload
````
## Restreindre SSH à un sous-réseau spécifique :
Limitez l’accès SSH à votre réseau local uniquement (par exemple, 192.168.x.x).
````powershell
[root@localhost ~]# firewall-cmd --zone=public --add-rich-rule="rule family='ipv4' service name='ssh' source address='192.168.1.0/24' accept" --permanent
[root@localhost ~]# firewall-cmd --reload
````