# BackUp System Borg

This tutorial shows how to set up a fully automatic, secure and incremental back up system on a Debian or Ubuntu server using Borg. Borg is used as the underlying technology.

## Preparation: 

* Set up Debian or Ubuntu server as a BackUp server.
* Best to rent this server from another server provider, or have it at your home. In any case in another pyhsic place.

## Source Server:

```bash
sudo -i 
cd && ssh-keygen && cat ~/.ssh/id_rsa.pub
```

## BackUp Server:

```bash
sudo apt install borgbackup vim ncdu -y && sudo adduser borg # No root permissions!

su borg 
cd && mkdir .ssh

vim ~/.ssh/authorized_keys # Add the public key from the source server
command="borg serve --restrict-to-path /home/borg/backups/Server1 --append-only" # Insert this before the public key you just inserted

mkdir -p ~/backups/Server1


borg init --encryption=none ~/backups/Server1
borg config ~/backups/Server1 additional_free_space 10G
```

## Source Server:

```bash
apt install borgbackup vim ncdu -y
vim ~/backup.sh
```
##### backup.sh:
```bash
#!/bin/bash

# Dump all databases
mysqldump -u root --all-databases > all_databases.sql
# To restore a Single MySQL Database from a Full MySQL Dump:
# mysql -p -o database_name < all_databases.sql

# Get apt selection of packages:
/usr/bin/dpkg --get-selections | /usr/bin/awk '!/deinstall|purge|hold/'|/usr/bin/cut -f1 |/usr/bin/tr '\n' ' '  > installed-packages.txt  2>&1
# To restore apt packages:
# sudo apt update && sudo xargs apt install </root/installed-packages.txt

DATE=`date +"%Y-%m-%d"`
REPOSITORY="ssh://borg@1.2.3.4:22/~/backups/Server1"
borg create --exclude-caches --one-file-system $REPOSITORY::$DATE / -e /dev -e /prox -e /sys -e /tmp -e /run -e /media -e /mnt

# Alternative run, if server says "Connection closed by remote host. Is borg working on the server?" but borg is definitely installed at the target server. 
#borg create --remote-path /usr/local/bin/borg --exclude-caches --one-file-system $REPOSITORY::$DATE / -e /dev -e /prox -e /sys -e /tmp -e /run -e /media -e /mnt
```
   
    

```bash   
chmod 700 ~/backup.sh && ~/backup.sh # Testing

crontab -e
0 2 * * * /root/backup.sh # daily at 2 am
```

### Create restore files:
```bash
vim ~/mount_backup.sh && chmod 700 ~/mount_backup.sh && vim ~/umount_backup.sh && chmod 700 ~/umount_backup.sh
```

```bash
#!/bin/bash

REPOSITORY="ssh://borg@1.2.3.4:22/~/backups/Server1"
borg mount $REPOSITORY /mnt

# Alternative run, if server says "Connection closed by remote host. Is borg working on the server?" but borg is definitely installed at the target server. 
#borg mount --remote-path /usr/local/bin/borg $REPOSITORY /mnt

echo "You can find your backups in /mnt. To copy files execute e.g. 'cp -ra /mnt/1970-01-01/var/* /var'"
echo "Please don't forget to umount your backups with '~/umount_backup.sh' afterwards."
```

```bash
#!/bin/bash

borg umount /mnt
```

## Backup Server

```bash
vim ~/prune-backup.sh
```
##### prune-backup.sh
```bash
#!/bin/bash
borg prune -v ~/backups/Server1 \
    --keep-daily=10 \
    --keep-weekly=6 \
    --keep-monthly=12
```

```bash
chmod 700 ~/prune-backup.sh && ~/prune-backup.sh # Testing

crontab -e
0 9 * * * /home/borg/prune-backup.sh # daily at 9 am
```


# How to restore a complete server:
```bash
# Start on a fresh server as root
apt update && apt dist-upgrade
apt install borgbackup
borg mount ssh://borg@1.2.3.4:22/~/backups/Server1 /mnt
cp /mnt/1970-01-01/root/installed-packages.txt /root/
xargs apt install <installed-packages.txt -y
reboot
borg mount ssh://borg@1.2.3.4:22/~/backups/Server1 /mnt
cp -ra /mnt/1970-01-01/var/* /var/
cp -ra /mnt/1970-01-01/etc/* /etc/
cp -ra /mnt/1970-01-01/opt/* /opt/
cp -ra /mnt/1970-01-01/home/* /home/
cp -ra /mnt/1970-01-01/root/* /root/
cp -ra /mnt/1970-01-01/var/* /var/
reboot
# Afterwards you have to renew the ssh certifcates at the backup server for the automatic backup
