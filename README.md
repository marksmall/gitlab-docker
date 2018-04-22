# GitLab Dockerization

* [Structure](#structure)
* [GitLab Front-end](#gitLab-front-end)
  * [Ports](#ports)
  * [Volumes](#volumes)
  * [Environment](#environment)
  * [HTTPS](#https)
* [Redis](#redis)
* [Postgres](#postgres)
* [GitLab CI Runner](#gitlab-ci-runner)
* [Backups](#backups)
  * [Backup Docker Container](#backup-docker-container)
  * [Docker Container Recovery](#docker-container-recovery)

This README file is intended to explain the reasons for the options chosen in the `docker-compose.yml` file.

## Structure

GitLab is split across **4 containers**:

* GitLab
* Redis
* Postgres
* GitLab Runner

All **Services** restart when shutdown, you may want to consider using `unless-stopped` instead though.

## GitLab Front-end

Th **GitLab service** looks quite daunting, but in reality, all the complexity resides in passing **GitLab** configuration options. **NOTE:** These configuration options are not added to `/etc/gitlab/gitlab.rb`, if you need to lookup these options you have to look at the `docker-compose.yml` file. We will look at the config options after we have first considered the rest of the service's options.

### Ports

I've configured the ports to use expose the default **HTTP** and **SSH** ports as I run local versions of these services and they clashed, a **Production** environment would not require this.

> **NOTE:** What would it need then?

### Volumes

A number of directories are persisted via **Named Volumes**. The config, logs, data and SSL related files we want to persist across software upgrades.

### Environment

This is where the complexity lies, but really it's not. It assigns the whole section to the _Environment variable_ **GITLAB_OMNIBUS_CONFIG**. These are **key/value** pairs that you would see in the `/etc/gitlab/gitlab.rb` file. Putting the config here allows it to persist between containers. The options set cover topics such as:

* HTTPS
* Container Registry
* GitLab Pages
* Log rotation
* Disabling **Redis** and **Postgres**, these are services in their own right.
* LDAP (commented out as I don't have the exact details to configure it. This would be the best way for normal users, it doesn't exclude us creating local user for external people)

## HTTPS

**HTTPS** is already configured in the example `docker-compose.yml` file. First off, we need a **certificate**, create a self-signed on using: `openssl req -x509 -newkey rsa:4096 -keyout gitlab.example.com.key -out gitlab.example.com.crt --nodes`. The hostname should match the hostname you will use. **NGINX** needs reference to these certificates, so move the key and certificate to the location set in the **volumes** section.

## Redis

The **Redis service** is very simple, therefore it runs within an **Alpine image**. It has no state to maintain, the job queue is transient, so if it goes down, any existing jobs will be lost.

## Postgres

The **Postgres service** is again quite simple and again uses an **Alipine image**, it maintains the software only. The data is is maintained in a **Named Volume**, so it can be shared between multiple services or just persisted beyond the lifetime of the service. Consider upgrading, you want the data to persist, but upgrade the software.

This service takes **Environment Variables** to configure secret details e.g. _username_ and _password_.

## GitLab CI Runner

The **GitLab CI Runner service** uses a **non-alpine** image, rather it uses the latest gitlab runner image. The configuration and the `/var/run/docker.sock` file **must** come before the configuration directory if you want to spawn runners via docker.

**NOTE:** This service I've not had a chance to test as I only have a local setup available, but it should work, the problems I ran into where DNS related as the **hostname** is only set in my `/etc/hosts` file.

## Backups

It is work backing up the containers and possibly even the locally mounted data, I'll focus on the containers.

## Backup Docker Container

Create a new **Docker image** using the command: `docker commit -p 0a6b63ba0739 gitlab-$(date +"%Y-%m-%d")` replace the **Container ID** with the real container id, you can get that by running: `docker ps`. The `commit` command pauses the running container and takes a snapshot of it, to create the image.

If you have a **Docker Repository** you can push the new image for re-deployment, if necessary. First login to the repository: `docker login`, then push the container to the registry: `docker push gitlab-<date>`.

You can also backup the image to a **.tar** file: `docker save -o ~/gitlab-<date>.tar gitlab-<date>`.

## Docker Container Recovery

If you pushed the image to a repository, you can just reference the backed up image in your `Dockerfile` or `docker-compose.yml` files. If it was backed up to a **.tar** file, you can **load the image** using: `docker load -i ~/gitlab-<date>.tar` and reference the image as before.
