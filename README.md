# How to configure GitLab on a Synology DSM6 using Docker

## The context

The purpose is to offer all DSM services only to the internal network and to serve the GitLab instance on the Internet.

## Configuring GitLab and it's container

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

## Configuring GitLab Runner
