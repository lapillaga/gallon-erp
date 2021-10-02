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

## Steps to Install Traefik Stack

### Create .env-trafik file

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

### Check Stack

Check if the stack was deployed with `docker stack ps traefik`

### Traefik logs

You can check the Traefik logs with:
```
docker service logs traefik_traefik
```

## Steps to Install MariaDb Stack

### Create a docker config

Go to `cd configs/`

Run `docker config create frappe-mariadb-config ./frappe-mariadb-config.config`

### Create a docker secret with db credentials

Create a folder secrets with `mkdir secrets` then create a file called `frappe-mariadb-root-password.config` with this content:
```
yourdatabasesupersecretpassword
```
Run `docker secret create frappe-mariadb-root-password ./frappe-mariadb-root-password.config`

### Run Maria DB Stack

Go to stacks directory and run `docker stack deploy --compose-file frappe-mariadb.yml frappe-mariadb`

### Inspect if all services have been created 

`docker stack services frappe-mariadb`

## Steps to Install ERPNext

If you will execute all commands as root user you must change docker volumes permission to `chown -R 1000:1000 /var/lib/docker/volumes`

### Create .env-frappe in stack folder
```
export ERPNEXT_VERSION=v13.11.1
export FRAPPE_VERSION=v13.11.0
export MARIADB_HOST=frappe-mariadb_mariadb-master
export SITES=\`erp.domain.com\`
```

### Export env variables

Run `source .env-frappe`

### Run Frappe Stack

Go to stacks folder and then run `docker stack deploy --compose-file frappe-bench-v13.yml frappe-bench-v13`

## Steps to create Site

Actually only works enter into docker container and executing 
```
bench new-site erp.domain.com --install-app erpnext --db-type mariadb --no-mariadb-socket --admin-password ADMIN_INITIAL_PASSWORD
```

## Steps to install custom apps
Go to custom-apps directory and change Dockerfile on custom nginx and worker. If root directory does not has execute permission you must run `chmod -R 777 .`.

### Build custom nginx docker image
```
docker build --build-arg=FRAPPE_BRANCH=v13.11.0 -t gallon-erpnext-nginx:v13.11.1 custom-apps/nginx
```

### Build custom worker docker image
```
docker build --build-arg=FRAPPE_BRANCH=v13.11.0 -t gallon-erpnext-worker:v13.11.1 custom-apps/worker
```

### Run Stack again after delete existing stack.


# Site operations

Create and use env file to pass environment variables to containers,

```sh
source .env
```

Or specify environment variables instead of passing secrets as command arguments. Refer notes section for environment variables required

## Setup New Site

Note:

- Wait for the database service to start before trying to create a new site.
    - If new site creation fails, retry after the MariaDB container is up and running.
    - If you're using a managed database instance, make sure that the database is running before setting up a new site.

#### MariaDB Site

```sh
# Create ERPNext site
docker run \
    -e "SITE_NAME=$SITE_NAME" \
    -e "DB_ROOT_USER=$DB_ROOT_USER" \
    -e "MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD" \
    -e "ADMIN_PASSWORD=$ADMIN_PASSWORD" \
    -e "INSTALL_APPS=erpnext" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    frappe/erpnext-worker:$VERSION new
```

#### PostgreSQL Site

PostgreSQL is only available v12 onwards. It is NOT available for ERPNext.
It is available as part of `frappe/erpnext-worker`. It inherits from `frappe/frappe-worker`.

```sh
# Create ERPNext site
docker run \
    -e "SITE_NAME=$SITE_NAME" \
    -e "DB_ROOT_USER=$DB_ROOT_USER" \
    -e "POSTGRES_HOST=$POSTGRES_HOST" \
    -e "POSTGRES_PASSWORD=$POSTGRES_PASSWORD" \
    -e "ADMIN_PASSWORD=$ADMIN_PASSWORD" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    frappe/erpnext-worker:$VERSION new
```

Environment Variables needed:

- `SITE_NAME`: name of the new site to create. Site name is domain name that resolves. e.g. `erp.example.com` or `erp.localhost`.
- `DB_ROOT_USER`: MariaDB/PostgreSQL Root user.
- `MYSQL_ROOT_PASSWORD`: In case of the MariaDB docker container use the one set in `MYSQL_ROOT_PASSWORD` in previous steps. In case of a managed database use the appropriate password.
- `MYSQL_ROOT_PASSWORD_FILE` - When the MariaDB root password is stored using docker secrets.
- `ADMIN_PASSWORD`: set the administrator password for the new site.
- `ADMIN_PASSWORD_FILE`: set the administrator password for the new site using docker secrets.
- `INSTALL_APPS=erpnext`: available only in erpnext-worker and erpnext containers (or other containers with custom apps). Installs ERPNext (and/or the specified apps, comma-delinieated) on this new site.
- `FORCE=1`: optional variable which force installation of the same site.

Environment Variables for PostgreSQL only:

- `POSTGRES_HOST`: host for PostgreSQL server
- `POSTGRES_PASSWORD`: Password for `postgres`. The database root user.

Notes:

- To setup existing frappe-bench deployment with default database as PostgreSQL edit the common_site_config.json and set `db_host` to PostgreSQL hostname and `db_port` to PostgreSQL port.
- To create new frappe-bench deployment with default database as PostgreSQL use `POSTGRES_HOST` and `DB_PORT` environment variables in `erpnext-python` service instead of `MARIADB_HOST`

## Add sites to proxy

Change `SITES` variable to the list of sites created encapsulated in backtick and separated by comma with no space. e.g. ``SITES=`site1.example.com`,`site2.example.com` ``.

Reload variables with following command.

```sh
docker-compose --project-name <project-name> up -d
```

## Backup Sites

Environment Variables

- `SITES` is list of sites separated by `:` colon to migrate. e.g. `SITES=site1.domain.com` or `SITES=site1.domain.com:site2.domain.com` By default all sites in bench will be backed up.
- `WITH_FILES` if set to 1, it will backup user-uploaded files.
- By default `backup` takes mariadb dump and gzips it. Example file, `20200325_221230-test_localhost-database.sql.gz`
- If `WITH_FILES` is set then it will also backup public and private files of each site as uncompressed tarball. Example files, `20200325_221230-test_localhost-files.tar` and `20200325_221230-test_localhost-private-files.tar`
- All the files generated by backup are placed at volume location `sites-vol:/{site-name}/private/backups/*`

```sh
docker run \
    -e "SITES=site1.domain.com:site2.domain.com" \
    -e "WITH_FILES=1" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    frappe/erpnext-worker:$VERSION backup
```

The backup will be available in the `sites-vol` volume.

## Push backup to s3 compatible storage

Environment Variables

- `BUCKET_NAME`, Required to set bucket created on S3 compatible storage.
- `REGION`, Required to set region for S3 compatible storage.
- `ACCESS_KEY_ID`, Required to set access key.
- `SECRET_ACCESS_KEY`, Required to set secret access key.
- `ENDPOINT_URL`, Required to set URL of S3 compatible storage.
- `BUCKET_DIR`, Required to set directory in bucket where sites from this deployment will be backed up.
- `BACKUP_LIMIT`, Optionally set this to limit number of backups in bucket directory. Defaults to 3.

```sh
 docker run \
    -e "BUCKET_NAME=backups" \
    -e "REGION=region" \
    -e "ACCESS_KEY_ID=access_id_from_provider" \
    -e "SECRET_ACCESS_KEY=secret_access_from_provider" \
    -e "ENDPOINT_URL=https://region.storage-provider.com" \
    -e "BUCKET_DIR=frappe-bench" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    frappe/frappe-worker:$VERSION push-backup
```

Note:

- Above example will backup files in bucket called `backup` at location `frappe-bench-v13/site.name.com/DATE_TIME/DATE_TIME-site_name_com-{filetype}.{extension}`,
- example DATE_TIME: 20200325_042020.
- example filetype: database, files or private-files
- example extension: sql.gz or tar

## Restore backups

Environment Variables

- `MYSQL_ROOT_PASSWORD` or `MYSQL_ROOT_PASSWORD_FILE`(when using docker secrets), Required to restore mariadb backups.
- `BUCKET_NAME`, Required to set bucket created on S3 compatible storage.
- `ACCESS_KEY_ID`, Required to set access key.
- `SECRET_ACCESS_KEY`, Required to set secret access key.
- `ENDPOINT_URL`, Required to set URL of S3 compatible storage.
- `REGION`, Required to set region for s3 compatible storage.
- `BUCKET_DIR`, Required to set directory in bucket where sites from this deployment will be backed up.

```sh
docker run \
    -e "MYSQL_ROOT_PASSWORD=admin" \
    -e "BUCKET_NAME=backups" \
    -e "REGION=region" \
    -e "ACCESS_KEY_ID=access_id_from_provider" \
    -e "SECRET_ACCESS_KEY=secret_access_from_provider" \
    -e "ENDPOINT_URL=https://region.storage-provider.com" \
    -e "BUCKET_DIR=frappe-bench" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    -v ./backups:/home/frappe/backups \
    --network <project-name>_default \
    frappe/frappe-worker:$VERSION restore-backup
```

Note:

- Volume must be mounted at location `/home/frappe/backups` for restoring sites
- If no backup files are found in volume, it will use s3 credentials to pull backups
- Backup structure for mounted volume or downloaded from s3:
    - /home/frappe/backups
        - site1.domain.com
            - 20200420_162000
                - 20200420_162000-site1_domain_com-*
        - site2.domain.com
            - 20200420_162000
                - 20200420_162000-site2_domain_com-*

## Edit configs

Editing config manually might be required in some cases,
one such case is to use Amazon RDS (or any other DBaaS).
For full instructions, refer to the [wiki](https://github.com/frappe/frappe/wiki/Using-Frappe-with-Amazon-RDS-(or-any-other-DBaaS)). Common question can be found in Issues and on forum.

`common_site_config.json` or `site_config.json` from `sites-vol` volume has to be edited using following command:

```sh
docker run \
    -it \
    -v <project-name>_sites-vol:/sites \
    alpine vi /sites/common_site_config.json
```

Instead of `alpine` use any image of your choice.

## Health check

For socketio and gunicorn service ping the hostname:port and that will be sufficient. For workers and scheduler, there is a command that needs to be executed.

```shell
docker exec -it <project-name>_erpnext-worker-d \
docker-entrypoint.sh doctor  -p postgresql:5432 --ping-service mongodb:27017
```

Additional services can be pinged as part of health check with option `-p` or `--ping-service`.

This check ensures that given service should be connected along with services in common_site_config.json.
If connection to service(s) fails, the command fails with exit code 1.

## Frappe internal commands using bench helper

To execute commands using bench helper.

```shell
 docker run \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    --user frappe \
    frappe/frappe-worker:$VERSION bench --help
```

Example command to clear cache

```shell
 docker run \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    --user frappe \
    frappe/frappe-worker:$VERSION bench --site erp.mysite.com clear-cache
```

Notes:

- Use it to install/uninstall custom apps, add system manager user, etc.
- To run the command as non root user add the command option `--user frappe`.


## Delete/Drop Site

#### MariaDB Site

```sh
# Delete/Drop ERPNext site
docker run \
    -e "SITE_NAME=$SITE_NAME" \
    -e "DB_ROOT_USER=$DB_ROOT_USER" \
    -e "MYSQL_ROOT_PASSWORD=$MYSQL_ROOT_PASSWORD" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    frappe/erpnext-worker:$VERSION drop
```

#### PostgreSQL Site

```sh
# Delete/Drop ERPNext site
docker run \
    -e "SITE_NAME=$SITE_NAME" \
    -e "DB_ROOT_USER=$DB_ROOT_USER" \
    -e "POSTGRES_PASSWORD=$POSTGRES_PASSWORD" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    --network <project-name>_default \
    frappe/erpnext-worker:$VERSION drop
```

Environment Variables needed:

- `SITE_NAME`: name of the site to be deleted. Site name is domain name that resolves. e.g. `erp.example.com` or `erp.localhost`.
- `DB_ROOT_USER`: MariaDB/PostgreSQL Root user.
- `MYSQL_ROOT_PASSWORD`: Root User password for MariaDB.
- `FORCE=1`: optional variable which force deletion of the same site.
- `NO_BACKUP=1`: option variable to skip the process of taking backup before deleting the site.

Environment Variables for PostgreSQL only:

- `POSTGRES_PASSWORD`: Password for `postgres`. The database root user.

## Migrate Site

```sh
# Migrate ERPNext site
docker run \
    -e "MAINTENANCE_MODE=1" \
    -v <project-name>_sites-vol:/home/frappe/frappe-bench/sites \
    -v <project-name>_assets-vol:/home/frappe/frappe-bench/sites/assets \
    --network <project-name>_default \
    frappe/erpnext-worker:$ERPNEXT_VERSION migrate
```

Environment Variables needed:

- `MAINTENANCE_MODE`: If set to `1`, this will ensure the bench is switched to maintenance mode during migration.
- `SITES`: Optional list of sites to be migrated, separated by colon (`:`). e.g. `erp.site1.com:erp.site2.com`. If not used, all sites will be migrated by default.
