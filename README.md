[![build docker image](https://github.com/indigo-iam/egi-trust-anchors-container/actions/workflows/build-docker-image.yml/badge.svg)](https://github.com/indigo-iam/egi-trust-anchors-container/actions/workflows/build-docker-image.yml)

# EGI trust anchors container

This is [container](https://hub.docker.com/r/indigoiam/egi-trustanchors) contains fetch-crl and other utilities
to provide up-to-date trust anchors to relying applications, like Nginx, and to VOMS Attribute Authority service. When run,
the container updates 2 components:

- The system trust anchors, known as CA bundle, typically found in /etc/pki on an EL-based system
- The EGI trust anchors maintained by fetch-crl and typically found in /etc/grid-security/certificates

These 2 components are exported as 2 separate volumes mapped when the container is launched and used as volumes read by other
containers needing these informations.

The container is currently based on AlmaLinux 9.


## Configuration

The main configuration needed is the declaration of the 2 volumes. To avoid that the volumes are destroyed when the container is
deleted, and thus become unavailable to the containers using them, it is recommended to create the volumes before launching
the containers and to declare the volumes as `external` in the Docker/Podman configuration. It is typically done with (volume names are
free but must be consistent across the configuration):

```bash
# podman command can be used if using Podman instead of Docker
docker volume create cabundle
docker volume create trustanchors
```

It is also necessary to define the following environment variable when launching the container so that fetch-crl is run as startup:

```
FORCE_TRUST_ANCHORS_UPDATE=1
```


## Compose file

Wheter using Docker or Podman, it is recommended to use a compose file to declare the container and how to run it. A typical
compose section for running this container would be:

```yaml
services:
    trustanchors:
        image: docker.io/indigoiam/egi-trustanchors:main
        container_name: egi-trustanchors
        environment:
            - FORCE_TRUST_ANCHORS_UPDATE=1
        volumes:
                - trustanchors:/etc/grid-security/certificates
                - cabundle:/etc/pki

volumes:
    cabundle:
        external: true
    trustanchors:
        external: true


```

## Cron job

The container exits after updating the trust anchors so it is necessary to add a cron job that periodically restarts the container
(typically once a day). If the container has been created at startup, for example by executing the compose file, it is enough to
restart the container with an entry like:

```
# podman command can be used if using Podman instead of Docker
0 3 * * * root (date --iso-8601=seconds --utc; (/usr/bin/docker restart egi-trustanchors)) >> /var/log/egi-trustanchors-update.log 2>&1
```

`Note: the log is basically 2 lines per run if no error occurs.`
