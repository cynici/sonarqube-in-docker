# Run Sonarqube with Docker

This repo spins up the community edition of this static code
analysis tool quickly and painlessly. It can process different
languages based on the *extensions* you install,
<https://www.sonarqube.org/features/multi-languages/>

---
**CAVEAT**

Sonarqube requires the database to use **UTF-8** encoding
hence the explicit `POSTGRES_INITDB_ARGS` environment variable
in [docker-compose.yml](docker-compose.yml)

I had assumed that alone was sufficient and had originally
used `postgis` image for the database server.

Strangely, Sonarqube started up fine the first time but
after installing extensions, restart would fail with a cryptic
error about a certain column not supporting UTF-8 encoding.

Sonarqube also has this warning displayed on
[Docker Hub](https://hub.docker.com/_/sonarqube/)

> Only a single instance of SonarQube can connect to a database schema. If you're using a Docker Swarm or Kubernetes, make sure that multiple SonarQube instances are never running on the same database schema simultaneously. This will cause SonarQube to behave unpredictably and data will be corrupted. There is no safeguard until [SONAR-10362](https://jira.sonarsource.com/browse/SONAR-10362).

---

The [docker-compose.yml](docker-compose.yml) mounts the following
directories as persistent storage for the containers in the
`volumes` standza.

You may specify alternative source path for your setup but the
target mount points must remain the same.

- `./pgdata` PostgreSQL data directory
- `./data`, `./logs`, `./extension` are used by Sonarqube,
  <https://docs.sonarqube.org/latest/setup/install-server/>

## Prerequisites

- Hefty hardware requirements,
  <https://docs.sonarqube.org/latest/requirements/requirements/>

- docker-ce, <https://docs.docker.com/engine/install/>

- docker-compose, <https://docs.docker.com/compose/install/>

- you are a member of `docker` group, so you can run `docker` CLI


## How to spin up fresh instance of Sonarqube

### Increase process limits

Create `/etc/sysctl.d/99-sonarqube.conf`

```
vm.max_map_count=262144
fs.file-max=65536
```

Apply the changes

```
sudo sysctl -p /etc/sysctl.d/99-sonarqube.conf
```

### Define environment variables

Create `.env`

Beware that the format may look familar but it is not shell-like.
Do not use quotes unless you mean it literally,
<https://docs.docker.com/compose/env-file/#syntax-rules>

This file is not intentionally [excluded from the repo](.gitignore)

```
POSTGRES_USER=sonarqube
POSTGRES_PASSWORD=You_should_change_this!
POSTGRES_DATABASE=sonarqube

POSTGRES_HOST=postgresql
POSTGRES_PORT=5432

POSTGRES_UID=1000
POSTGRES_GID=1000

SQ_IMAGE=sonarqube:8-community

SQ_HTTP_PORT=80
```

- `POSTGRES_PASSWORD` this is the plaintext password.
  `POSTGRES_USER`, `POSTGRES_DATABASE` may be set to
  something else if you prefer but they must not be changed
  after PostgreSQL and Sonarqube have run for the first time
  and having gotten the database initialized.
  If you have to change any of these after initialization,
  you need to know enough PostgreSQL administrative commands
  to alter the existing database accordingly.
  Additional environment variables to customize the
  `postgeres` docker image, <https://hub.docker.com/_/postgres>

- `POSTGRES_UID`, `POSTGRES_GID` may be omitted completely.
  I set them to my UID/GID so that I will be the owner of
  entire PostgreSQL data directory tree so I can edit, copy
  stuff around without requiring `sudo` privilege outside
  the container. The `postgres` image actually doesn't
  honor these variables. They are used by the docker
  entrypoint script [set-postgres-uid.sh](set-postgres-uid.sh)

- `POSTGRES_HOST`, `POSTGRES_PORT` should not be changed if you
  are using the example [docker-compose.yml](docker-compose.yml)
  in this repo exactly where the PostgreSQL database server
  and Sonarqube containers share the same private docker network.
  `POSTGRES_HOST` value is set to match the name of PostgreSQL
  service defined in [docker-compose.yml](docker-compose.yml)
  If you run PostgreSQL server elsewhere, set these accordingly.
  
- `SQ_IMAGE` is the Sonarqube community edition release that
  I have tested to work. I don't know if future or past releases
  also work with the setup given in this repo.

- `SQ_HTTP_PORT` is the tcp port that the Sonarqube dashboard
  can be accessed on the docker host. If you run this service
  over a network even if private, I highly recommend that you
  add a `nginx` service to serve as the TLS-enabled reverse proxy.
  Then you remove the `ports` stanza completely from docker-compose.yml
  and configure nginx to use `sonarqube:9000` as a HTTP upstream
  <https://medium.com/@pentacent/nginx-and-lets-encrypt-with-docker-in-less-than-5-minutes-b4b8a60d3a71>

Validate your `.env` file. The output should be full YAML file with
the correct values substituted, without any warning message

```
docker-compose config
```

The Sonarqube docker image may be customized with extra environment variables,
<https://docs.sonarqube.org/latest/setup/environment-variables/>

## How to add projects to Sonarqube

Here, I describe what I find to be the quickest way - run Sonarqube
CLI scanner against a repository already present on your workstation.

For production use involving multiple teams or large number of
projects (code repo), you probably want to consider more scaleable
and effective approaches, preferrably one that integrates into
your CI/CD workflow:

- the official documentation mentions Jenkins and GitLab
  <https://docs.sonarqube.org/latest/analysis/branch-pr-analysis-overview/>

- using Sonarqube dashboard,
  <https://docs.bitnami.com/oci/how-to/analyze-projects-sonarqube/>


### Pull images, initilize database and application

You only need to do this once.

The datbase directory must be completely empty otherwise the
bootstrapping script will not perform database initialization.

```
# Pull required images
docker-compose pull

# Start only the database service in background
docker-compose up -d postgresql

# 'tail -f' the container log
docker-compose logs -f postgresql
```

From the scrolling log output, you should see a new database being created.
When the output shows `database system is ready to accept connections`
and stops scrolling, press CTRL-C and stop watching the log.

Next, start Sonarqube container which will detect the fresh
instance is being created. This process may take up to 3 minutes.

```
docker-compose up -d

docker-compose logs -f
```

When the output stops scrolling and shows `... SonarQube is up`,
press CTRL-C to stop watching the log. The application is ready!


### Change default password and create user

- Go to http://{dockerhost}/ 

- Sonarqube default administrator username and password are both `admin`

- Click on the user icon in the top right corner - My Account

- Go to Security tab and change the password

### Get a token

- In the same My Account - Security tab, click "Generate toke"

- Copy and save token safely somewhere

### Add language extensions

- Navigate to Marketplace

- Install the extensions you want. Typically, it will prompt
  you to restart Sonarqube. So you prabably want to choose
  a few of them at one go.

### Import project using scanner

Run the CLI scanner against a source directory to import
it as a new project in Sonarqube. The time the scanner takes
depends on the size of the source tree.

This example below uses your repo directory name as the
Sonarqube project name. If you want a different project name,
modify `projectKey` value accordingly.

You may pass additional project parameters in the same command,
<https://docs.sonarqube.org/latest/analysis/analysis-parameters/>

Based on the language extensions you have installed,
Sonarqube automatically detects them based on the file names.
You can explicitly exclude specific file/directory from being
scanned:

- <https://stackoverflow.com/a/35358747>
- <https://docs.sonarqube.org/latest/analysis/coverage/>

```
cd {your_repo_directory}

docker run -it \
 -e SONAR_TOKEN={your_token} \
 -e SONAR_HOST_URL=http://{dockerhost} \
 -v $PWD:/usr/src \
 sonarsource/sonar-scanner-cli sonar-scanner \
 -Dsonar.projectKey=$(basename $PWD)
```

Further information about the scanner:

- <https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/>
- <https://github.com/SonarSource/sonar-scanner-cli-docker/>
