# Steps to deploy in docker swarm

## Requirements
- Install docker and docker-compose.
- Create new user and add to docker and sudo group.
- It's necesary configure a dns record on your provider. In the example case it will be `erp.domain.com`
- Init docker swarm mode with `docker swarm init`

## Folder structure
    .
    ├── configs
    │   ├── frappe-mariadb-config      # Maria DB initial docker config
    ├── stacks                    # Test files (alternatively `spec` or `tests`)
    │   ├── frappe-mariadb.yml          # Docker compose Maria DB services
    │   ├── traefik.yml         # Proxy service and SSL Certificates
    │   └── frappe-bench-v13.yml                # Docker compose file to execute frappe-erp-next services
    └── ...

## Steps to Install Traefik Service

### Start traefik.yml service

Go to `cd stacks/` then create a `.env-traefik` file with the following content:
```
export NODE_ID=$(docker info -f '{{.Swarm.NodeID}}')
export EMAIL=email@example.com
export DOMAIN=traefik.domain.com
export USERNAME=username
export PASSWORD=supersecretpassword
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
```

### Execute `.env-traefik`

Run `source .env-traefik`

### Creat a docker network to traefik-public

Run `docker network create --driver=overlay traefik-public`. This command create a network that will be shared with Traefik and the containers that should be accessible from the outside.

### Create a tag

Create a tag in this node, created as env variable in last step, so that Traefik is always deployed to the same node and uses the same volume:

`docker node update --label-add traefik-public.traefik-public-certificates=true $NODE_ID`

### Run Stack

`docker stack deploy -c traefik.yml traefik`


### Create a file config called `frappe-mariadb-config`

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
