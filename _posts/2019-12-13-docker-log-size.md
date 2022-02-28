---
layout: post
title: "Docker error : no space left on device"
author: "João Neto"
categories: journal
tags: [docker,container,ec2,aws,log]
---

Yesterday morning I was informed the [KNoT](https://knot-devel.cesar.org.br/) cloud instance, which we run in EC2 machines on AWS, were down. We don't have robust monitoring services yet and I usually enter the cluster manager instance to discover why the services are down and to bring them up again.

The first thing I did was to inspect one of the stopped containers and I saw the error "no space left on device".

```bash
docker service ps <container_id> --no-trunc

ld6rde7xuv2pwsg8270b7qo21    \_ authenticator  <image>   <address>    Shutdown   Rejected <time>   "symlink <path> <overlay>: no space left on device"
```

After that, maybe the obvious action to take is to analyze the instance's disk space:

```bash
df -h

/dev/xvda1       16G  8.8G  7.3G  55% /
```

As we can see, the disk space isn't the problem but the error is clearly indicating it's a space problem. So, after searching quickly the docker overlay directories became the main target and I noticed the container log files were too big for the expected overlay size. For example, one of these files had 2.2G of size.

```bash
sudo du -d1 -h /var/lib/docker/containers | sort -h

2.2G	/var/lib/docker/containers/c07ca3a6f88c0b88718c7...
```

To solve this problem I followed these steps:

1. Decide if the log files must be truncated or entirely cleared.

    - Truncate: a better option when you are concerned about the containers that are running at the moment.

        ```bash
        sudo sh -c "truncate -s 0 /var/lib/docker/containers/*/*-json.log"
        ```

    - Clear: a better option you aren't concerned about the state of the running containers.

        ```bash
        cat /dev/null > /var/lib/docker/containers/*/*-json.log`
        ```

1. Update the docker configuration file to set limits on log size and enable it to be rotated. Besides,
I also set a limit on the number of files stored after rotation.

    To do this, edit the file `/etc/sysconfig/docker` and modify the `OPTIONS` variable to add these flags: `--log-opt max-size=<size> --log-opt max-file=<number>`

1. Restart the docker service.

    ```bash
    systemctl restart docker
    ```

This solution worked very well and all the services are now running properly.

You can find more information about how to configure docker logging related options [here](https://docs.docker.com/config/containers/logging/json-file/
) and [here](https://docs.docker.com/config/containers/logging/configure/).


References:

<https://stackoverflow.com/questions/42510002/how-to-clear-the-logs-properly-for-a-docker-container>

<https://access.redhat.com/solutions/2334181>

<https://success.docker.com/article/no-space-left-on-device-error>

<https://medium.com/@Quigley_Ja/rotating-docker-logs-keeping-your-overlay-folder-small-40cfa2155412>