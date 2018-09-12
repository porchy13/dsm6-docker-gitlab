# How to configure a private integration server on a Synology DSM6 using Docker

This tutorial aims to install GitLab and SonarQube to create a private integration server on a Synology based on an Intel chip.

The purpose is to offer all DSM services only on the internal network and to serve the GitLab instance over the Internet.

The Runners will use Docker to perform the CI operations. This tutorial is focused on Java projects that are build using Maven.

This tutorial implies that you have a SSH access to your Synology with an admin user. The ```docker``` and ```docker-compose``` commands have to be executed using ```sudo```.

One unique _docker-compose.yml_ file will be created and is available in the repository. All chapters will only present their part of the global Docker configuration. All services will run on a unique network called _cicdnet_. So, all urls will directly match the container name.

> ⚠ When connecting to a container, it's name can be different as the one shown in this article.

## Docker images and plugins

GitLab will be configured to serve trough the context _gitlab_. Using the GitLab embedded Nginx instance, SonarQube will serve it's pages through the context _sonarqube_. The root will automatically been redirect to _gitlab_.

All services will be secured using a Let's Encrypt certificate. This certificate will be automatically obtain using GitLab adequate support. The usage of the HTTPS protocol is mandatory to bind GitLab and SonarQube.

The following docker images will be used:
* PostgreSQL: [_postgres:9.6-alpine_](https://store.docker.com/images/postgres)
* GitLab Enterprise Edition: [_gitlab/gitlab-ee:latest_](https://store.docker.com/community/images/gitlab/gitlab-ee)
* SonarQube: [_sonarqube:alpine_](https://store.docker.com/images/sonarqube)
* GitLab Runners: [_gitlab/gitlab-runner:latest_](https://store.docker.com/community/images/gitlab/gitlab-runner)
* WatchTower: [_v2tec/watchtower_](https://store.docker.com/community/images/v2tec/watchtower)

Additionally, some plugins will be used for SonarQube:
* [SonarQube GitLab Plugin](https://gitlab.talanlabs.com/gabriel-allaigre/sonar-gitlab-plugin) to integrate the SonarQube reports to the commits directly into GitLab.
* [SonarQube Gitlab Auth Plugin](https://gitlab.talanlabs.com/gabriel-allaigre/sonar-auth-gitlab-plugin) to authenticate users into SonarQube using GitLab.

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

To get the sub-net to limit access to the databases, the value can be obtained by inspecting the network.

```shell
$ sudo docker network ls
$ sudo docker network inspect <network_name>
```
To finish the setup, the container should be restarted.

## Configuring GitLab and it's container

The configuration shown below retrieves the last version of GitLab Enterprise Edition. The ports are mapped to ports that will be exposed to the Internet.

> ⚠ GitLab depends on SonarQube to start. This is due to Nginx that check the existance of the SonarQube container before launching.

```yaml
gitlab:
  image: 'gitlab/gitlab-ee:latest'
  restart: always
  hostname: '<url>'
  privileged: true
  depends_on:
    - postgres
    - sonarqube
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

The configuration portion of GitLab shown below only focused on the changed lines:
* GitLab will be accessible through the context _gitlab_.
* The e-mail setting is not shown, take a look at the GitLab configuration pages for more details.
* The embedded instance of PostgreSQL will be deactivated and the connection to the PostgreSQL container is made through the container's name.
* The communications will only pass through HTTPS and the access to the root will be automatically redirect to the _gitlab_ context.
* The SSL certificate will be obtain through Let's Encrypt.

```Ruby
# [...]
external_url 'https://<url>/gitlab'

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
gitlab_rails['db_host'] = "postgres"

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

nginx['custom_gitlab_server_config'] = "
location = / {
  return 302 https://$host/gitlab/;
}"

# [...]
################################################################################
# Let's Encrypt integration
################################################################################
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['<your-email>']
```

After GitLab has been reconfigured and restarted (see commands below), open the GitLab site and create the _root_ password. After that, configure GitLab as you want.

```shell
$ gitlab-ctl reconfigure
$ gitlab-ctl restart
```

## Configuring SonarQube and it's container

The SonarQube configuration is shown below. The database password is set into an _.env_ file located at the same place of the _docker-compose.yml_ file. It only contains ```SONARQUBE_PASSWORD=secret```. It will be accessed through the context _sonarqube_.

```yaml
sonarqube:
  image: 'sonarqube:alpine'
  restart: always
  depends_on:
    - postgres
  command: -Dsonar.web.context=/sonarqube
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

To serve SonarQube, the embedded instance of Nginx in GitLab should redirect the corresponding traffic to the corresponding context. The _gitlab.rb_ should be adapted as shown below:

```conf
nginx['custom_gitlab_server_config'] = "
location = / {
  return 302 https://$host/gitlab/;
}

location /sonarqube/ {
  proxy_pass              http://sonarqube:9000;

  proxy_set_header        Host                    $host;
  proxy_set_header        X-Real-IP               $remote_addr;
  proxy_set_header        X-Forwarded-For         $proxy_add_x_forwarded_for;
}"
```

After the modifications, GitLab should be reconfigured and only Nginx should be restarted:

```shell
$ gitlab-ctl reconfigure
$ gitlab-ctl restart nginx
```

When the container runs, the first thing to do is to change the _admin_ password. By default, it is set to _admin_. Before installing the needed extensions for GitLab, there are some options that can be set (at your convenience) in the _Administration_ menu:
* _Configuration_ > _Security_ > _Force user authentication_: yes
* _Configuration_ > _General_ > _Server base URL_: https://&lt;url&gt;/sonarqube
* _Security_ > _Permission Templates_ > _Default template_ > _Project Creators_: check _Administer_

Now, let's install and configure the GitLab related plugins. In the _Marketplace_, search for "gitlab" plugins and mark for install _GitLab_ and _GitLab Auth_. Do not forgive to also check the updatable plugins and select them to restart SonarQube only once.

After the restart, a new _GitLab_ section is shown in the _Administration_ menu. Select it and configure the plugins as following:
* Authentication
  * _Enabled_: yes
  * _GitLab url_: https://&lt;url&gt;/gitlab
  * _Application ID_: get the value from _GitLab_ > _Admin area_ > _Applications_ > _New application_ with the following data:
    * _Name_: SonarQube
    * _Redirect URI_: https://&lt;url&gt;/sonarqube/oauth2/callback/gitlab
    * _Trusted_: checked
    * _Scopes_: read_user
  * _Secret_: as above
  * _Allow users to sign-up_: yes
  * _Synchronize user groups_: yes
  * _User exceptions_: root
* Reporting
  * _GitLab url_: https://&lt;url&gt;/gitlab
  * _Comment when no new issue_: yes
  * _All issues_: yes
  * _Load rules information_: yes

After the configuration has been done, SonarQube must be restarted.

## Configuring GitLab Runner

The various runners will be executed through the dedicated container configured below:

```yaml
gitlab-runner:
  image: 'gitlab/gitlab-runner:latest'
  restart: always
  depends_on:
    - gitlab
  networks:
    - cicdnet
  volumes:
    - '/var/run/docker.sock:/var/run/docker.sock'
    - gitlab-runner_conf:/etc/gitlab-runner
```

After it has been started, the necessary runners should be configured like shown below. This example creates a runner running into a Docker container that contains the Maven environnement.

```shell
$ sudo docker exec -ti gitlab-runner bash
gitlab-runner$ gitlab-runner register
Running in system-mode.
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/): https://<url>/gitlab/
Please enter the gitlab-ci token for this runner: <from_gitlab>
Please enter the gitlab-ci description for this runner: maven3-jdk8
Please enter the gitlab-ci tags for this runner (comma separated): maven3-jdk8
Registering runner... succeeded
Please enter the executor: docker
Please enter the default Docker image (e.g. ruby:2.1): maven:3-jdk-8-alpine
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```

> To cache Maven or Ivy artifacts, the runners configuration should be changed in the file _/etc/gitlab-runner/config.toml_ of the container (or in the corresponding volume). For each runner concerned, the _volumes_ directive should be changed to add the volume that will contain the artifacts:

```toml
volumes = ["/cache","m2_cache:/root/.m2"]
# or
volumes = ["/cache","ivy2_cache:/root/.ivy2"]
```

To persist the corresponding volumes, add them to the _docker-compose.yml_ file.

## Configuring WatchTower

WatchTower will update automatically the containers. The definition below will configure WatchTower to check for updates once a week. You can change this by changing the ```command``` directive by using [Cron expression](https://godoc.org/github.com/robfig/cron#hdr-CRON_Expression_Format).

```yaml
watchtower:
  image: v2tec/watchtower
  restart: always
  networks:
    - cicdnet
  environment:
    - WATCHTOWER_NOTIFICATIONS=email
    - WATCHTOWER_NOTIFICATION_EMAIL_FROM=${EMAIL_FROM}
    - WATCHTOWER_NOTIFICATION_EMAIL_TO=${WATCHTOWER_TO}
    - WATCHTOWER_NOTIFICATION_EMAIL_SERVER=${EMAIL_SMTP}
    - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PORT=${EMAIL_SMTP_PORT}
    - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_USER=${EMAIL_USERNAME}
    - WATCHTOWER_NOTIFICATION_EMAIL_SERVER_PASSWORD=${EMAIL_PASSWORD}
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  command: --schedule @weekly
```

It has been chosen to be notified by email when an update occurs. The various email variables will be stored in the _.env_ file (see (#configuring-sonarqube-and-its-container)).

> ⚠ If you experience problems by not receiving mails on a secure connection, check for this answer on [StackOverflow](https://stackoverflow.com/a/11664176).

## The DSM Firewall

As the services are completely configured and running, it's time to configure the Firewall of the DSM (Control Panel > Security > Firewall) to let the services be accessible from the internet. To create the corresponding rule, choose the Firewall profile and click on _Edit Rules_. Create a new rule using the following configuration:
* Ports
  * Custom (and click on the _Custom_ button)
    * _Type_: Destination port
    * _Protocol_: TCP
    * _Ports (separate with commas)_: 60022,60080,60443
* Source IP
  * All
* Action
  * Allow

> ⚠ Do not select a _built-in applications_ while creating the rule. If you stop all service in one time using ```docker-compose stop```, the rule will be deactivated.

## References

- https://docs.gitlab.com/ee/install/docker.html
- https://store.docker.com/images/maven
- https://store.docker.com/images/postgres
- https://store.docker.com/images/sonarqube
- https://stackoverflow.com/questions/46753336/docker-compose-and-postgres-official-image-environment-variables?rq=1
- https://docs.gitlab.com/runner/install/docker.html
- https://store.docker.com/images/maven
- https://store.docker.com/community/images/hseeberger/scala-sbt
- https://github.com/v2tec/watchtower
- https://stackoverflow.com/a/11664176
- https://godoc.org/github.com/robfig/cron#hdr-CRON_Expression_Format
