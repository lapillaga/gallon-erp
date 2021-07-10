# Steps to deploy in docker swarm

## Create a repo with necesary files 

## Init docker swarm `docker swarm init`

## Create a file config called `frappe-mariadb-config`

## Run `docker config create frappe-mariadb-config ./frappe-mariadb-config`

## Creat a secret with `docker secret create frappe-mariadb-root-password ./frappe-mariadb-root-password`

## Then go to stacks directory and run `docker stack deploy --compose-file frappe-mariadb.yml frappe-mariadb`

## Inspect if all services have been created `docker stack services frappe-mariadb`
