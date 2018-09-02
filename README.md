# How to configure a private integration server on a Synology DSM6 using Docker

This tutorial aims to install GitLab and SonarQube to create a private integration server on a Synology based on an Intel chip.

The purpose is to offer all DSM services only on the internal network and to serve the GitLab instance over the Internet.

The Runners will use Docker to perform the CI operations. As it is written below, this tutorial is focused on Java projects that are build using Maven.

This tutorial implies that you have a SSH access to your Synology with an admin user. The ```docker``` and ```docker-compose``` commands have to be executed using ```sudo```.

One unique _docker-compose.yml_ file will be created and is available in the repository. All chapters will only present their part of the global Docker configuration. All services will run on a unique network called _cicdnet_. So, all urls will directly match the container name.

> ⚠ When connecting to a container, it's name can be different as the one shown in this article.

## Configuring PostgreSQL and it's container

PostgreSQL will be shared between GitLab and SonarQube as an independent piece. The version 9.6 will be used to avoid problems with GitLab:
* The official docker image of PostgreSQL do not allow to export binaries as a volume (it refuse to start if set).
* PostgreSQL binaries shipped with GitLab are from version 9.6 and they do not allow to connect to a 10+ PostgreSQL version.

The connection to PostgreSQL will be done through the specific network. The port won't be exposed outside. If a maintenance task should be done, use the ```docker exec ...``` command.

```yaml
postgres:
  image: 'postgres:9.6-alpine'
  restart: always
  networks:
    - cicdnet
  volumes:
    - postgresql:/var/lib/postgresql
    - postgresql_data:/var/lib/postgresql/data
```

The next step is to set the password for the super-admin, creates the databases for GitLab and SonarQube and define how they can be accessed. Let's start with the password:

```shell
$ sudo docker exec -ti postgres_1 bash
bash-4.4# su - postgres
postgres:~$ psql -c "alter user postgres with encrypted password 'secret'"
ALTER ROLE
```

Now, let's create the databases and the corresponding users. The commands below are for GitLab:
```shell
postgres:~$ createuser gitlab
postgres:~$ createdb -O gitlab gitlab
postgres:~$ psql -c "alter user gitlab with encrypted password 'secret'"
ALTER ROLE
postgres:~$ psql -d gitlab -c "create extension pg_trgm"
CREATE EXTENSION
```

And the commands below are for SonarQube: 

```shell
postgres:~$ createuser sonar
postgres:~$ createdb -O sonar sonar
postgres:~$ psql -c "alter user sonar with encrypted password 'secret'"
ALTER ROLE
```

The last thing is to change the file _pg_hba.conf_, so each user can only have access to it's database, except for user _postgres_ that should have access to all databases. Every uncommented lines should be commented or suppressed to avoid any side effect with the directives that will be added:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                md5    
host    gitlab          gitlab          172.20.0.0/16           md5    
host    sonar           sonar           172.20.0.0/16           md5
```

To finish the setup, the container should be restarted.

## Configuring GitLab and it's container

The configuration shown below retrieves the last version of GitLab Enterprise Edition. The ports are mapped to ports that will be exposed to the Internet.

```yaml
gitlab:
  image: 'gitlab/gitlab-ee:latest'
  restart: always
  hostname: 'your-host.com'
  privileged: true
  depends_on:
    - postgres
  ports:
    - '60080:80'
    - '60443:443'
    - '60022:22'
  networks:
    - cicdnet
  volumes:
    - gitlab_conf:/etc/gitlab
    - gitlab_log:/var/log/gitlab
    - gitlab_data:/var/opt/gitlab
```

The GitLab configuration file is updated directly into the container to have access to the ```gitlab-ctl``` command and have the full log of the _reconfigure_ stage:

```shell
$ sudo docker-compose pull
$ sudo docker-compose up -d
$ sudo docker exec -ti gitlab_gitlab_1 bash
root@host:/# vi /etc/gitlab/gitlab.rb
# see below how to configure
root@host:/# gitlab-ctl reconfigure
```

The configuration portion of GitLab shown below only focused on the changed lines. Remember that the connection to the PostgreSQL container is made through the socket. The e-mail setting is not shown, take a look at the GitLab configuration pages for more details.

```Ruby
# [...]
external_url 'https://<hostname>'

# [...]
################################################################################
## gitlab.yml configuration
##! Docs: https://gitlab.com/gitlab-org/omnibus-gitlab/blob/master/doc/settings/gitlab.yml.md
################################################################################
gitlab_rails['time_zone'] = '<your timezone>'

### Email Settings
# [...] everything needed to send e-mails

# [...]

### GitLab database settings
###! Docs: https://docs.gitlab.com/omnibus/settings/database.html
###! **Only needed if you use an external database.**
gitlab_rails['db_adapter'] = "postgresql"
gitlab_rails['db_encoding'] = "unicode"
gitlab_rails['db_database'] = "gitlab"
gitlab_rails['db_username'] = "gitlab"
gitlab_rails['db_password'] = "secret"
gitlab_rails['db_host'] = "/var/run/postgresql/"

# [...]

################################################################
## GitLab PostgreSQL
################################################################

###! Changing any of these settings requires a restart of postgresql.
###! By default, reconfigure reloads postgresql if it is running. If you
###! change any of these settings, be sure to run `gitlab-ctl restart postgresql`
###! after reconfigure in order for the changes to take effect.
postgresql['enable'] = false

# [...]

################################################################################
## GitLab NGINX
##! Docs: https://docs.gitlab.com/omnibus/settings/nginx.html
################################################################################
# [...]
# To have Let's Encrypt fully functional, this parameter has to be set to true
nginx['redirect_http_to_https'] = true

# [...]
################################################################################
# Let's Encrypt integration
################################################################################
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['<your-email>']
```

After GitLab has been reconfigured, open the GitLab site and create the _root_ password. After that, configure GitLab as you want.

## Configuring SonarQube and it's container

The SonarQube configuration is shown below. The database password is set through an _.env_ file located at the same place of the _docker-compose.yml_ file. It only contains ```SONARQUBE_PASSWORD=secret```. It is not planned to give access to SonarQube from the Internet. It will be only available from the private network.

```yaml
sonarqube:
  image: 'sonarqube:alpine'
  restart: always
  depends_on:
    - postgres
    - gitlab
  ports:
    - '60081:9000'
  networks:
    - cicdnet
  environment:
    - SONARQUBE_JDBC_URL=jdbc:postgresql://postgres:5432/sonar
    - SONARQUBE_JDBC_USERNAME=sonar
    - SONARQUBE_JDBC_PASSWORD=${SONARQUBE_PASSWORD}
  volumes:
    - sonarqube_conf:/opt/sonarqube/conf
    - sonarqube_data:/opt/sonarqube/data
    - sonarqube_extensions:/opt/sonarqube/extensions
    - sonarqube_bundled-plugins:/opt/sonarqube/lib/bundled-plugins
```

When the container runs, the first thing to do is to change the _admin_ password. By default, it is set to _admin_. Before installing the needed extensions for GitLab, there are some options that can be set (at your convenience) in the _Administration_ menu:
* _Configuration_ > _Security_ > _Force user authentication_ > yes
* _Security_ > _Permission Templates_ > _Default template_ > _Project Creators_: check _Administer_

Now, let's install and configure the GitLab related plugins. In the _Marketplace_, search for "gitlab" plugins and mark for install _GitLab_ and _GitLab Auth_. Do not forgive to also check the updatable plugins and select them to restart SonarQube only once.

After the restart, a new _GitLab_ section is shown in the _Administration_ menu. Select it and configure the plugins as following:
* Authentication
  * _Enabled_: yes
  * _GitLab url_: <your_public_url>
  * _Application ID_: get the value from _GitLab_ > _Admin area_ > _Applications_ > _New application_ with the following data:
    * _Name_: SonarQube
    * _Redirect URI_: http://<your_private_url>:60081
    * _Trusted_: checked
    * _Scopes_: read_user
  * _Secret_: as above
  * _Allow users to sign-up_: yes
  * _Synchronize user groups_: yes
  * _User exceptions_: root
* Reporting
  * _GitLab url_: <your_public_url>
  * _Comment when no new issue_: yes
  * _All issues_: yes
  * _Load rules information_: yes

After the configuration has been done, SonarQube must be restarted.

## The DSM Firewall

The first 

## Configuring GitLab Runner

## Updating the containers

Before updating a container, it is safe to stop it. The commands below stop all containers before pulling the new images. You can focus your update to only one container but you have to be sure of what you are doing, the order of operation has to be maintained.

For example, if there are updates of GitLab **and** PostgreSQL, they could not be done at the same time. GitLab needs to update it's database and PostgreSQL must be running during this step.

```shell
$ sudo docker-compose -f <path_to>/nexus3.yml stop
$ sudo docker-compose -f <path_to>/sonarqube.yml stop
$ sudo docker-compose -f <path_to>/gitlab.yml stop
$ sudo docker-compose -f <path_to>/gitlab-runner.yml stop
$ sudo docker-compose -f <path_to>/postgresql.yml stop
```

```shell
$ sudo docker-compose -f <path_to>/nexus3.yml pull
$ sudo docker-compose -f <path_to>/sonarqube.yml pull
$ sudo docker-compose -f <path_to>/gitlab.yml pull
$ sudo docker-compose -f <path_to>/gitlab-runner.yml pull
$ sudo docker-compose -f <path_to>/postgresql.yml pull
```

```shell
$ TODO
```

## References

https://docs.gitlab.com/ee/install/docker.html

https://store.docker.com/images/maven

https://store.docker.com/images/postgres

https://store.docker.com/images/sonarqube

https://stackoverflow.com/questions/46753336/docker-compose-and-postgres-official-image-environment-variables?rq=1