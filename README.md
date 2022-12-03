```
       ____                  __          ____          .-.
      / __ \____  _____ ____/ /_  ____  / / /_    .--./ /      _.---.,
     / /_/ / __ `/ ___/ ___/ __ \/ __ \/ / __/     '-,  (__..-`       \
    / ____/ /_/ (__  |__  ) /_/ / /_/ / / /_          \                |
   /_/    \__,_/____/____/_,___/\____/_/\__/           `,.__.   ^___.-/
                                                         `-./ .'...--`
```

# Migrating passbolt source install to container
You may ask yourself: Why do I need to look into this documentation if there's already an existing offical one on [Passbolt's Website](https://help.passbolt.com/hosting/upgrade/pro/migrate-existing-pro-to-docker.html)?  
The manual is incomplete. Numerous services, which are a prerequisite for successful use in a corporate environment, are not configured. The manual does not include certificates (SSL, LDAPs), no mail dispatch, assumes certain scenarios (e.g. subscription key in database) and many more.  
So I hope I can be able to help you with my documentation!

## Requirements
- your current passbolt source install server
- and a new server with docker and docker-compose
- [official docker image from docker hub](https://hub.docker.com/r/passbolt/passbolt/) depending on your server edition (non-root-ce, non-root-pro, ce, pro)
- btrfs storage drivers in case you want to use btrfs for your passbolt container

## Backup of the current state
This chapter is about exporting and saving the essential files in order to implement them later during the actual migration.

### Backing up the pgp-key  
```
/etc/passbolt/gpg/serverkey.asc
/etc/passbolt/gpg/serverkey_private.asc
```
### License  
Depending in your setup, the license-key is either in the database or exists as a txt-file on your server
```
/etc/passbolt/subscription_key.txt
```
### Certificates  
- ssl-cert of your webserver
Optional (in case of using LDAPS or self-signed certs):
- intermediate cert
- root ca
- ldaps cert

Usually located in:
```
/etc/ssl/certs
```
If you can't find them, look into the ssl configuration of your webserver.  

### Database  
Create a sql dump of your passbolt database
```
mysqldump -u [user] -p passbolt > [filename].sql
```

That's all for now, we will need all these files later on during the migration.  

## Migrating to Docker using docker-compose
Go ahead and create the paths according to your needs, you can use the docker-compose.yml-file I added to the project  
Pay attention to change the following according to your setup and needs:
- image-version of MariaDB
- env_file of MariaDB
- location of mysqldump
- image-version of passbolt
- env_file of passbolt
- location of certificates, licensekey, gpg and passbolt.php
- and of course also contained names and volumes

### Restoring the database
When the container is started, the database is initialized using the environment variables provided. In addition, all files of the extension .sh and .sql, both in uncompressed and compressed form (gz, xz, zst), which are located or included in the entrypoint directory at the time of the "compose up" command, are loaded.  
There is a script in the container that performs this task:  
```
/usr/local/bin/docker-entrypoint.sh
```
[MariaDB documentation on Docker Hub](https://hub.docker.com/_/mariadb)  

### License and serverkeys
The license is either in the database or in the subscription_key.txt on your server. So it's either already finished by migrating the database or you can mount it in a volume in the compose script together with the serverkeys and the rest.  

## Starting containers
Without `-d` to watch migrating steps and watch out for possible errors. Docker will continue to pull all images, set up the containers, volumes as well as executing the `/docker-entrypoint.sh` script to import the mysql dump etc.
```
docker-compose up
```

## Healthchecks
### ... using cake
```
su -s /bin/bash -c "./bin/cake passbolt healthcheck" www-data
```
### ... using your browser
```
https://<fqdn>/healthcheck
```
 
## Testing
Make sure to test the fuctionatilty of your passbolt server to confirm the success of your migration
- successful healthcheck in cli and web
- ldaps-sync
- creating, syncing, shareing and copying passwords accross your server
- account recovery
