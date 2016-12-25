+++
date = "2015-11-09T20:37:49-05:00"
title = "Sync git repos with a sidekick container"

+++

Most of what I do these days - configuration, code and even blog posts are stored in a git repository somewhere.
With the addition of [Large File Storage] on Github, I've also started storing some binary files in Git.

Often, I have to save the content of a repo in a Docker container. How to do this reliably, especially without rebuilding a container that needs to access the content of the repo? 

I've created a small [Docker container](https://github.com/groob/docker-git-sync) to do just that. I can specify a repo, a branch, a destination and optionaly even a revision and the container will sync the repo to disk. Large File Storage is built in the container by default to take advantage of git-lfs as well. 

```bash
docker run --rm \
    -e GIT_SYNC_REPO=https://github.com/groob/docker-git-sync.git \
    GIT_SYNC_BRANCH=master \
    GIT_SYNC_DEST=/data \
    GIT_SYNC_REV=d95545407ecc1a5707111c77fe6ebae6011327f9 \
    groob/git-sync
```

A clever use for such a container is to run it as a "sidekick" along with your primary container. For example, the nginx container, recommends that we add static html content with 
`docker run --name some-nginx -v /some/content:/usr/share/nginx/html:ro -d nginx`

Instead of storing the html content in /some/content directory on local disk, we can create a volume using docker.
First, lets create an empty volume. This volume will persist on disk until we delete every container that mounts the volume. 
`
docker run --name nginx.volume -v /usr/share/nginx/html busybox /bin/true
`
Now, we can sync the content we want nginx to use with `groob/git-sync`

```bash
docker run --name nginx-sync --rm \
    -e GIT_SYNC_REPO=https://my_munki_repo.git \
    -e GIT_SYNC_DEST=/usr/share/nginx/html \
    --volumes-from nginx.volume \
    groob/git-sync
```
Once the content syncs, we can make it available with nginx:

```bash
docker run -d --name nginx p 80:80 --volumes-from nginx.volume nginx
```

I use a similar sidekick to sync the content of this blog. One of the great featues of [docker volumes](http://docs.docker.com/v1.8/userguide/dockervolumes/) is that they can be mounted by multiple containers that need to share the same data. 
We can utilize this pattern to sync configuration to public images hosted on Docker Hub.

# Pods
[Kubernetes](http://kubernetes.io/) uses pods, rather than container as it's most basic unit. A pod is a combination of one or more contaienrs that run together as a single atomic unit. Grouping the configuration and server containers as I've shown above is one example of a Pod. There is a proposal to make pods a default docker feature. You can read the proposal and discussion [here](https://github.com/docker/docker/issues/8781).
