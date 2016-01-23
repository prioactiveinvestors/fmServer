# FM server schema


![alt tag](https://github.com/prioactiveinvestors/composeFM/blob/master/schema.png)


**the compose project uses env and endProd to setup each container.
these are not tracked with git**

```
CREATE THE MACHINES - same options just different machine name
  docker-machine create \
      --driver generic \
      --generic-ip-address xxxxxxxxxxxx \
      --generic-ssh-key=/Users/krasimir/.ssh/id_rsa \
      --generic-ssh-user root \
      --engine-opt cluster-store=consul://xxxxxxxxxxxx:8500 \
      --engine-opt cluster-advertise=xxxxxxxxxxxx:2376 \
      --engine-opt cluster-store-opt=kv.cacertfile=/etc/docker/ca.pem \
      --engine-opt cluster-store-opt=kv.certfile=/etc/docker/server.pem \
      --engine-opt cluster-store-opt=kv.keyfile=/etc/docker/server-key.pem \ 
      machineName
  
  mount ram drive on both machines for the nginx pagespeed cache
  nano /etc/fstab
  tmpfs /home/fm/http/ngx_pagespeed_cache tmpfs size=1024m,mode=0777,uid=root,gid=nogroup 0 0

PROD SERVERS
  **first server** 
  (for initial deployment need to bootstrap new mysql so uncomment #MYSQL_PRIMARY=1 in the env file)
    eval $(docker-machine env fm-first)
    docker-compose -f first-prod.yml build
    docker-compose --x-networking --x-network-driver overlay -f first-prod.yml up -d
    
    docker-compose -f first-always-on-prod.yml build
    docker-compose --x-networking --x-network-driver overlay -f first-always-on-prod.yml   up -d
  **second server**
    eval $(docker-machine env fm-second)
    docker-compose -f second-prod.yml --project-name second build
    docker-compose --x-networking --x-network-driver overlay -f second-prod.yml --project-name second  up -d

    docker-compose -f second-always-on-prod.yml --project-name second build
    docker-compose --x-networking --x-network-driver overlay -f second-always-on-prod.yml --project-name second  up -d
  **backup**
    eval $(docker-machine env   pai-backup)
    docker-compose -f backup-prod.yml --x-networking up -d consul
    docker-compose --x-networking --x-network-driver overlay -f backup-prod.yml  up -d garbd backup-manager

DEV SERVERS

```

NOTES
  currently only the first server runs cron jobs so if it is down than the cron jobs need be started on the second server
  if the second_mariadb doesn't start delete .sst and sst_in_progress files
  when adding new node it might refuse to start if the master is in recover state. on master run RESET MASTER ; and then start the node to be added. alternatevly try specifing the run command as : mysqld --tc-heuristic-recover=commit

  the backup agents don't run in a container
  the backup manager is looking for the mysql data files in /var/lib/mysql (as this is the location reported by the mysql server) 
    so we need ln -s /home/fm/mysql /var/lib/mysql
  if the automated agent isntallation fails use 
    cd /lib/modules/r1soft/ && wget -c http://repo.r1soft.com/modules/Debian_8_x64/hcpdriver-cki-3.16.0-4-amd64.ko && /etc/init.d/cdp-agent restart 


  chmod  -R 777 /home/fm/http/fullertreacymoney.com/chart/public/app/webroot/thumbs
  chmod  -R 777 /home/fm/http/fullertreacymoney.com/chart/public/app/webroot/upload
  chmod  -R 777 /home/fm/http/fullertreacymoney.com/chart/public/app/tmp/
  chmod  -R 777 /home/fm/http/fullertreacymoney.com/www/public/system/data

  chown  -R team:nogroup /home/fm/http/fullertreacymoney.com/chart/public/
  chown  -R team:nogroup /home/fm/http/fullertreacymoney.com/www/public/



SETUP LOCAL DEV

```
...projects
          fm
              server
                  compose
                  docker
              mysql
              http
                  default
                  fullertreacymoney.com
                  www
                  chart
          vipconsul
              docker
```

cd to ..../projects 
git clone ssh://root@51.255.67.13/home/fm/http/fullertreacymoney.com/www/git fm/http/fullertreacymoney.com/www
git clone ssh://root@51.255.67.13/home/fm/http/fullertreacymoney.com/chart/git fm/http/fullertreacymoney.com/chart
git clone https://github.com/prioactiveinvestors/fmServer.git fm/server
git clone https://github.com/vipconsult/dockerfiles.git vipconsult/docker

mkdir -p fm/http/fullertreacymoney.com/chart/public/app/tmp/cache/models
mkdir -p fm/http/fullertreacymoney.com/chart/public/app/tmp/cache/persistent
chmod -R 777 fm/http/fullertreacymoney.com/chart/public/app/tmp
chown -R YOURUSER projects

#add a hook to change permissions on every merge
```
  nano fm/http/fullertreacymoney.com/www/.git/hooks/post-merge
  #!/bin/sh
  sudo -R chmod 777 ./public/system/data
  sudo -R chown YOURUSER ./

  nano fm/http/fullertreacymoney.com/chart/.git/hooks/post-merge
  #!/bin/sh
  sudo chmod 777 -R ./public/app/webroot/thumbs
  sudo chmod 777 -R ./public/app/webroot/upload
  sudo chmod 777 -R ./public/app/tmp/
  sudo chown -R YOURUSER ./

```

sudo rsync -vza --stats --progress root@51.255.67.13:/home/fm/http/default/ fm/http/default

download the data for the mysql from the backup server
create the envDev file for compose

cd fm/server/compose
docker-compose --x-networking -f dev.yml up -d




