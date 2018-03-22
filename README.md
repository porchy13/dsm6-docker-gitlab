# How to configure GitLab on a Synology DSM6 using Docker

The purpose is to offer all DSM services only on the internal network and to serve the GitLab instance over the Internet.

The Runners will use Docker to perform the CI operations. As it is written below, this tutorial is focused on Java projects that are build using Maven.

This tutorial implies that you have a SSH access to your Synology with an admin user. The ```docker``` and ```docker-compose``` commands have to be executed using ```sudo```.

## Configuring GitLab and it's container

The container will be created and launched using ```docker-compose```, also to simplify the updates. The ```docker-compose.yml``` shown below retrieves the last version of GitLab Enterprise Edition. Important data is stored in the hard-drive of the Synology (and not in the image). The ports are mapped to ports that will be exposed to the Internet.

```YML
web:
  image: 'gitlab/gitlab-ee:latest'
  restart: always
  hostname: '<hostname>'

  ports:
    - '60080:80'
    - '60443:443'
    - '60022:22'

  volumes:
    - '/<path>/gitlab/config:/etc/gitlab'
    - '/<path>/gitlab/logs:/var/log/gitlab'
    - '/<path>/gitlab/data:/var/opt/gitlab'
```

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
# # [...] everything needed to send e-mails

# [...]
################################################################################
# Let's Encrypt integration
################################################################################
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['<your-email>']
```

## The DSM Firewall

The first 

## Configuring GitLab Runner

## References

https://docs.gitlab.com/ee/install/docker.html

https://store.docker.com/images/maven

https://store.docker.com/images/postgres

https://store.docker.com/images/sonarqube
