+++
date = "2015-05-15T07:40:28-04:00"
draft = false
title = "Certified - an internal CA for your company"

+++

Many of the services we set up today require SSL and it's useful to be able to generate/revoke certificates as you wish. This costs money, and the tools for generating your own CA are dificult to use. Recently I started using [certified](https://github.com/rcrowley/certified) which is a set of shell scripts to manage an internal CA. Certified is easy to use, and also comes in with a built in git repo for your CA. Whenever you generate, sign or revoke a certificate, it all gets tracked in a Git repo. 

I did not want to install any extra dependencies on my system, so instead, I packaged Certified in a [docker container](https://registry.hub.docker.com/u/groob/certified/). 
To get started, just pull the container from Docker hub.

```
docker pull groob/certified
``` 

Let's generate our CA.

```
$ mkdir certs
$ docker run --rm -it --name certified \
	-v $(pwd)/certs:/certified/etc \
	-e GIT_USER=me  \
	-e GIT_EMAIL=me@example.com \
	groob/certified \
	certified-ca C="US" ST="CA" L="San Francisco" O="Example" CN="Example CA"

certified-ca: generating 4096 bit keys for the CA but defaulting to 2048 for other certificates
certified-ca: the CA certificates will expire in 3650 days instead of 365
...
...
certified-ca: store /certified/etc/ssl/private/root-ca.key in a very safe place and remove it from here
``` 

Above, we created a 'certs' directory and mounted it into our docker container. Certified generated both a root-ca and a intermediate CA. As the instructions say, we will move our CA key to a safe place(I saved it to an encrypted disk image) and use the intermediate CA to sign our certificates.

Now, we'll sign a certificate for one of our applications:

```
$ docker run --rm -it --name certified \
	-v $(pwd)/certs:/certified/etc \
	-e GIT_USER=me  \
	-e GIT_EMAIL=me@example.com \
	groob/certified \
	certified CN="munki.example.com"
```

In the certs/ssl/certs directory you will find munki.example.com.crt and in the certs/ssl/private directory you will find munki.example.com.key. We can now add those to our nginx config.
First, let's bundle the intermediate CA cert and the munki cert together. 

```
$ cat munki.example.com.crt ca.crt > munki.crt
```

and add those to our nginx config

```
server {
    listen              443 ssl;
    server_name         munki.example.com;
    ssl_certificate     munki.crt;
    ssl_certificate_key munki.example.com.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
}
```

Our final step is to add the root CA cert to the System Keychain on our end user machines. 
To do that, I created a package with the postinstall script shown in [rtrouton's blog post](https://derflounder.wordpress.com/2011/03/13/adding-new-trusted-root-certificates-to-system-keychain/).

Once the root ca is on all our machines, any certificate we sign with the intermediary CA will be trusted. 

