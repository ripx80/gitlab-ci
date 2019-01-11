# gitlab-ci with docker

## Enable HTTPS

### Generate Certs

Look at the full [HowTo](https://github.com/GetchaDEAGLE/gitlab-https-docker/blob/master/README.md#generating-a-self-signed-certificate)

```bash
cd volume_gitlab/ssl
openssl genrsa -out server-key.pem 4096
sudo openssl req -new -key server-key.pem -out server.csr
openssl x509 -req -days 365 -in server.csr -signkey server-key.pem -out server-cert.pem
rm server.csr
```

### add this self-sign certs on your git profile

you must add this self sign cert to your local certs or in your git config.

nano ~/.gitconfig

    [http]
        sslCAinfo = /home/rip/orchestration/gitlab-ci/ssl/server-cert.pem

And if you will trust this cert system wide on Arch do this:

    sudo trust anchor server-cert.pem

test it. if you have not cert error, everything is fine

    curl https://gitlab.local:443

## mkcert

Or you use the mkcert tool for self signed certs. You only do this:

```bash
mkcert -install
mkcert github.local localhost ::1
```

## Create the dev net

    docker network create development

## End it up

    docker-compose up -d

Access your gitlab with a browser, add the cert to your trusted certs in browser store

    https://gitlab.local/

if you run this docker on localhost, add following to your /etc/hosts

    ::1             gitlab.local gitlab

yes its ipv6. suprise suprise the time is ticking...

## Start without docker-compose (dev)

```bash
docker run -d --name gitlab -p 0.0.0.0:443:443 -p 0.0.0.0:80:80 -p 0.0.0.0:5222:22 --restart always --env GITLAB_OMNIBUS_CONFIG="external_url 'https://gitlab.local'" \
--hostname gitlab.local \
--volume /mnt/store/docker/storage/gitlab/config:/etc/gitlab:z \
--volume /mnt/store/docker/storage/gitlab/logs:/var/log/gitlab:z \
--volume /mnt/store/docker/storage/gitlab/data:/var/opt/gitlab:z \
--volume /etc/localtime:/etc/localtime \
gitlab/gitlab-ce:latest
```

## Running in Development

You can run it without using volumes, only bind mounts in dev mode.

```bash
docker-compose -f docker-compose.yml -f dev.yml up -d
```

## Running in Production

Read more about stages with docker-compose [here](https://docs.docker.com/compose/production/#deploying-changes)
Build your own image with your ssl certs. change the tag if you will and not forget to change it in production.yml

```bash
docker build -t ripx80/gitlab-ce-prod:latest .
docker-compose -f docker-compose.yml -f prod.yml up -d
```

## Upgrade

    docker-compose pull && docker build -t ripx80/gitlab-ce-prod:latest . && docker-compose -f docker-compose.yml -f prod.yml up --force-recreate -d

## Backup with gitlab-rake (recommended)

You can find a howto [here](https://docs.gitlab.com/ee/raketasks/backup_restore.html)
The container name is gitlab in this example.

```bash
docker exec -t gitlab gitlab-rake gitlab:backup:create
docker exec -it gitlab ls /var/opt/gitlab/backups/
docker cp gitlab:/var/opt/gitlab/backups/1547206794_2019_01_11_11.6.3_gitlab_backup.tar #example tar
# or tar all with permissions
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar cvf /backup/gitlab-backups.tar /
```

But this does not store your configuration files. You may also want to back up any TLS keys and certificates
Backup these files form config dir:

- /etc/gitlab/gitlab-secrets.json
- /etc/gitlab/gitlab.rb

```bash
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar cvf /backup/config-backup.tar /
```

And if you want continous backups use your crontab.

```bash
# crontab -l
0 7 * * * /bin/docker exec -i gitlab gitlab-rake gitlab:backup:create CRON=1
0 7 * * * /bin/docker run --rm --volumes-from gitlab -v /storage/backup:/backup busybox tar cvf /backup/config-backup.tar /etc/gitlab CRON=1
```

## Restore with gitlab-rake

```bash
#copy backup to volume
docker cp $(pwd)/backup/1547208579_2019_01_11_11.6.3_gitlab_backup.tar gitlab:/var/opt/gitlab/backups/
# restore permissions
docker exec -t gitlab chown 998.998 /var/opt/gitlab/backups/1547208579_2019_01_11_11.6.3_gitlab_backup.tar

# or tar with permissions
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar xpvf /backup/gitlab-backups.tar -C /

# restore github backup
docker exec -it gitlab gitlab-rake gitlab:backup:restore BACKUP=1547208579_2019_01_11

#copy config to volume
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar xpvf /backup/config-backup.tar -C /

#restart container
docker-compose restart
```

## Backup with docker

```bash
mkdir backup
docker stop gitlab
docker commit gitlab backup/gitlab-container # create a new image
docker save backup/gitlab-container >backup/gitlab-container-backup.tar # save image to tar

# save volumes
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar cvf /backup/config-backup.tar /etc/gitlab
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar cvf /backup/data-backup.tar /var/opt/gitlab
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar cvf /backup/logs-backup.tar /var/log/gitlab
```

## Restore with docker

```bash
# if you need this special backup container load it as image and start it with the right name. Dont do this if you use the default container
docker load -i backup/gitlab-container-backup.tar

# create the default gitlab container and restore volumes
docker-compose -f docker-compose.yml -f prod.yml up --no-start
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar xpvf /backup/config-backup.tar -C /
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar xpvf /backup/data-backup.tar -C /
docker run --rm --volumes-from gitlab -v $(pwd)/backup:/backup busybox tar xpvf /backup/logs-backup.tar -C /
#check if all data on the right place
docker run -it --rm --volumes-from gitlab busybox
# now start it
docker-compose -f docker-compose.yml -f prod.yml up -d
```

## Restart gitlab

    docker exec -it gitlab gitlab-ctl restart