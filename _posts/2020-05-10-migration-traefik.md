---
layout: post
title: "Migrating services to Traefik v2"
author: "João Neto"
categories: journal
tags: [traefik,loadbalancer,knot,proxy,reverseproxy,docker]
---

In the KNoT project, we use traefik as a reverse proxy technology to route requests to the right service, which is running in a Docker container. Recently, traefik released its second version with the replacement of old components such as `frontends` and `backends` by `routers`, `middlewares`, and `services`.

To keep up-to-date with the newer traefik features and considering the addition of `routers` provides a better and flexible way of handling generic TCP requests, we decided to migrate our stack configuration for this new version. This note will briefly describe how this migration was done.

## Traefik v1

Firstly, let's introduce some traefik concepts from its **first version**:

* **entrypoints**: network entry points from which traefik is listening (port, SSL, redirection).
* **frontends**: matching patterns (based on request fields) that indicate the backend target (aka Docker service) of the request.
* **backends**: composed by one or more target services, which can include a load-balancing strategy.

Below you can see an example of traefik configuration that setup two `entrypoints`: one for listening `https` requests on 443 port and another for listening `http` requests on port 80 and automatically redirecting to the `https` entry point.

```yaml
  traefik:
    image: traefik:v1.7
    command: >
      traefik
        --docker
        --docker.watch
        --docker.swarmMode
        --docker.exposedByDefault=false
        --entryPoints='Name:http Address::80 Redirect.EntryPoint:https'
        --entryPoints='Name:https Address::443 TLS'
        --defaultEntryPoints=http,https
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      mode: global
```

After receiving the request through one of these entry points, traefik will route it according to the service's `frontend` rules. As an example, we have two services `storage` and `babeltower`, which expects the request according to the address domain defined by a regular expression that catches every request that starts with `storage.` or `bt.`.

```yaml
  babeltower:
    image: cesarbr/knot-babeltower
    env_file: '../env.d/knot-babeltower.env'
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.frontend.rule=HostRegexp:bt,bt.{domain:[a-zA-Z0-9.]+}
        - traefik.port=80
  storage:
    image: cesarbr/knot-cloud-storage
    env_file: '../env.d/knot-cloud-storage.env'
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.frontend.rule=HostRegexp:storage,storage.{domain:[a-zA-Z0-9.]+}
        - traefik.port=80
```

It's important to notice that when we define the `frontend` rules dynamically on Docker service's labels, it's already attached to the service `backend`. So, we don't need any additional configuration to link the `frontend` to the `backend`.

## Traefik v2

Now, there are no `frontends` or `backends` anymore. They were rewritten into two new components: `routers` and `services`. This will give us more flexibility because, differently of `backends`, the `services` aren't responsible for applying changes on the fly to the requests anymore, moving this role the `middlewares`. Then, it's possible to create one that rewrites pathnames and attach it to a particular `router`.

The traefik settings remains quite similar to the v1:

```yaml
  traefik:
    image: traefik:v2.2
    command: >
      traefik
        --providers.docker
        --providers.docker.watch
        --providers.docker.swarmMode
        --providers.docker.exposedbydefault=false
        --entrypoints.http.address=:80
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    deploy:
      mode: global
```

Based on the following steps, you can see the result after migrating the traefik version.

1. Specify a port for the `services` through its load balancer server labels.
1. Create a `router` rule that will enable matching a request hostname to the services based on a defined `regex`.
1. Attach the `router` to an `entrypoint`.

```yaml
  babeltower:
    image: cesarbr/knot-babeltower:dev
    env_file: './env.d/knot-babeltower.env'
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.services.babeltower.loadbalancer.server.port=80
        - traefik.http.routers.babeltower.rule=HostRegexp(`{subdomain:bt}.{domain:[a-zA-Z0-9.]+}`)
        - traefik.http.routers.babeltower.entrypoints=http
  storage:
    image: cesarbr/knot-cloud-storage:dev-go
    env_file: './env.d/knot-cloud-storage.env'
    deploy:
      replicas: 1
      labels:
        - traefik.enable=true
        - traefik.http.services.storage.loadbalancer.server.port=80
        - traefik.http.routers.storage.rule=HostRegexp(`{subdomain:storage}.{domain:[a-zA-Z0-9.]+}`)
        - traefik.http.routers.storage.entrypoints=http
```

Was it simple, right?

In the previous example, I didn't use any middleware, so how it could be useful? Hm. Exactly. We can setup a middleware for automatically redirect `http` requests to `https` ones.  This note will be further updated to add this example and all the necessary `https` configuration.

Click [here](https://github.com/CESARBR/knot-cloud/blob/master/stacks/core/prod/all-in-one/stage-1.yml) to see the full version of this stack (without `https` too).