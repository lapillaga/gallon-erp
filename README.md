# Steps to deploy in docker swarm

## Create a repo with necesary files 

## Init docker swarm `docker swarm init`

## Create a file config called `frappe-mariadb-config`

## Run `docker config create frappe-mariadb-config ./frappe-mariadb-config`

## Creat a secret with `docker secret create frappe-mariadb-root-password ./frappe-mariadb-root-password`

## Then go to stacks directory and run `docker stack deploy --compose-file frappe-mariadb.yml frappe-mariadb`

## Inspect if all services have been created `docker stack services frappe-mariadb`

## If you will execute all commands as root user you must change docker volumes permission to `chown -R 1000:1000 /var/lib/docker/volumes`

## Create .env in stack folder and then export with this content
```
ERPNEXT_VERSION=v13.6.0
FRAPPE_VERSION=v13.6.0
MARIADB_HOST=frappe-mariadb_mariadb-master
SITES=erp.elgallonegroec.com```

```

## Go to stacks folder and then run `docker stack deploy --compose-file frappe-bench-v13.yml frappe-bench-v13`
