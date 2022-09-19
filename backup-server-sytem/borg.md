# BackUp System Borg (mit Jeans Optimierungen)

In dieser Anleitung wird gezeigt, wie man auf einem Debian oder Ubuntu Server mit Borg ein vollautomatisches, sicheres und inkrementelles BackUp-System einrichtet. Als darunterliegende Technologie wird Borg verwendet.

## Vorbereitungen: 

* Debian oder Ubuntu Server als BackUp Server einrichten
* Am besten diesen Server bei einem anderen Serveranbieter mieten, oder bei sich Zuhause haben. Auf jeden Fall an einem anderen pyhsischen Ort.

## Source Server:

```bash
sudo -i && cd && ssh-keygen && cat ~/.ssh/id_rsa.pub
```

## BackUp Server:

```bash
sudo apt install borgbackup vim && sudo adduser serverbackup # Keine Root Rechte!

su serverbackup && cd && mkdir .ssh

vim ~/.ssh/authorized_keys # Den Public key vom Source Server hinzufügen
command="borg serve --restrict-to-path /home/serverbackup/backups/Server --append-only" # Das vor dem eben eingefügten Public Key einfügen

mkdir -p ~/backups/Server1

borg init --encryption=repokey ~/backups/Server1
borg key export ~/backups/Server1 ~/key-export && cat ~/key-export && rm ~/key-export # Diesen Schlüssel sicher aufbewahren und die Datei danach vom Server löschen.
```

## Source Serer:

```bash
sudo apt install borgbackup vim -y
vim ~/backup.sh
```
##### backup.sh:
```bash
#!/bin/bash

# Dump all databases
#mysqldump -u root --all-databases > all_databases.sql

# Restore a Single MySQL Database from a Full MySQL Dump:
# mysql --one-database database_name < all_databases.sql

DATE=`date +"%Y-%m-%d"`
REPOSITORY="ssh://serverbackup@1.2.3.4:22/~/backups/Server1"
export BORG_PASSPHRASE="MeineSuperSicherePassphrase"
borg create $REPOSITORY::$DATE /etc /home /opt /root /usr /var/www /var/lib /var/log--exclude-caches

# Alternative run, if server says "Connection closed by remote host. Is borg working on the server?" but borg is definitely installed at the target server. 
#borg create --remote-path /usr/local/bin/borg $REPOSITORY::$DATE /etc /home /opt /usr /var/www /var/lib /var/log --exclude-caches
```
   
    

```bash   
chmod +x ~/backup.sh && ~/backup.sh # Austesten

crontab -e
0 2 * * * /home/jean/backup.sh # Jeden Tag um 2:00 Uhr
```

## Backup Server

```bash
vim ~/prune-backup.sh
```
##### prune-backup.sh
```bash
#!/bin/bash
export BORG_PASSPHRASE="MeineSuperSicherePassphrase"
borg prune -v ~/backups/Server1 \
    --keep-daily=10 \
    --keep-weekly=6 \
    --keep-monthly=12
```

```bash
chmod +x ~/prune-backup.sh
~/prune-backup.sh # Austesten

crontab -e
0 9 * * * /home/serverbackup/prune-backup.sh # Jeden Tag um 9:00 Uhr
```