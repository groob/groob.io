+++
date = "2016-06-13T11:31:20-04:00"
draft = false
title = "Secure Munki server with Let's Encrypt and SCEP"

+++
In this post, I will show you how to set up a public Munki repository. The server will be configured to use HTTPS, but also require clients that connect to provide an X.509 client certificate to access the repo. Normally, this kind of setup would require your organization to pay for a SSL certificate, and set up a [PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure) system that will sign unique certificates for each device. Some enterprising MacAdmins have used the puppet CA to issue certificates for each Mac, but if you're not already using puppet, this option is less attractive. Today, I will show you how to configure a secure server, without any cost, and with very little technical knowledge necessary.

First, let's cover the technologies we'll use today:

* [Let's Encrypt](https://letsencrypt.org/) - A free certificate authority which will issue valid certificates for your site/subdomain. We will be using this service to get a certificate for `munki.groob.io`. This is optional if you want to use a self-signed certificate, or if your organization already pays a CA to have one signed for `munki.yourcompany.com`. Note that for LE to issue certificates, your server's DNS must resolve publicly. Since this is a demo for how to set up a public repo, this is great for us. Otherwise, we can always use our own certs in the config.
* [SCEP](https://tools.ietf.org/html/draft-gutmann-scep-02) - SCEP is a protocol that allows signing client cerificate requests. We will use it to issue certificates to our devices. This demo will use the [micromdm/scep](https://github.com/micromdm/scep) project that I've been developing recently. The project provides a SCEP server and a client which can request new certificates.
* [Caddy](https://caddyserver.com/) - Caddy is an http server, just like Apache or NGINX, which we need to host a munki repository. Caddy is great because it comes in with built in support for Let's Encrypt, but also because we can extend its functionality by writing custom plugins in [Go](https://golang.org/). We won't be using Caddy directly in this project, but it's important for you to know what it is.
* [Squirrel](https://github.com/micromdm/squirrel) - Squirrel is a custom version of the Caddy server, which adds specific features for deploying and managing munki repositories. Squirrel's first features are to allow deploying a secure Munki repo without too much hassle, but there are a lot more features planned, including a WebUI and API support.

## Important Note
Both the SCEP tools and `squirrel` are in preview. Over the next coming months I will be adding more features and fixing bugs. Please this treat this tutorial as a demo as well. 

## Setup
Now that we've covered the technologies used, let's deploy our Munki server.  
First, let's get the SCEP tools. You can download the latest version [here](https://github.com/micromdm/scep/releases). We'll be deploying the `scepclient` tool on every Mac, but we only need the server to create a CA.
```
scepserver ca -init -organization groob-io
```
The above command creates a CA that we will use to sign and validate *client* certificates. The tool will create a new folder called `depot` with a private key and CA cert:
```
depot
├── ca.key
└── ca.pem
```

Our devices will all need to trust this CA. There are two ways to do it. One is to add it to a profile and install the profile on each device. The second way is by adding the CA as a trusted root on each device. We'll handle that in the munki preflight script.

Now, let's configure our server. Caddy uses a configuration file named `Caddyfile` by convention, so that's what we'll call ours.
```
# Caddyfile
# See all valid configuration options here: https://caddyserver.com/docs/getting-started

# SCEP server configuration
# Note that the port for SCEP must be different from the server. 
# We use :2016, but any valid port will do.
https://scep.groob.io:2016 {
    log
    # The email is necessary for LE to send us expiration reminders
    tls groob@gmail.com
    # SCEP server configuration. 
    # Depot is the path to the depot folder we created earlier.
    # The challenge is a preshared password used by your server to validate requests from the client. 
    # Only clients that use the same challenge will be granted a certificate.
    scep {
        depot depot
        challenge secret
    }
}


# munki repo config
https://munki.groob.io {
    gzip
    log
    # same email here
    # also configure the server to request client certificates and validate them with the CA.
    tls groob@gmail.com {
        clients require depot/ca.pem
    }
    # set the root of your munki repo
    root /path/to/munki/repo
    browse /
}
```

Start the `squirrel` server.

```
# -agree is required for configuring LE
squirrel -agree -conf=/path/to/your/Caddyfile
```

The server will start up and serve your Munki repo. If you try to navigate to it in the browser, you'll be denied access. Our munki repo is secure.  
Now, let's request client certificates, and enable our devices to connect to the repo. We'll do these steps manually here, but you can automate these steps later.

```
# create the certs directory
sudo mkdir -p /Library/Managed\ Installs/certs/
# copy CA cert we created in the beggining to certs directory
sudo cp ca.pem /Library/Managed\ Installs/certs/
# create a private key and request a new certificate
sudo scepclient -private-key /Library/Managed\ Installs/certs/client.key -server-url=https://scep.groob.io:2016/scep -challenge=secret -cn=munki-client

# configure munki preferences
sudo defaults write /Library/Preferences/ManagedInstalls.plist ClientKeyPath "/Library/Managed Installs/certs/client.key"
sudo defaults write /Library/Preferences/ManagedInstalls.plist UseClientCertificate -bool true

# check that it's working
sudo managedsoftwareupdate -vvvv --checkonly --munkipkgsonly
```

Congratulations, you set up a secure Munki repo that only your munki clients can access.

If you're interested in the development of SCEP, Squirrel and MDM join me in the `#micromdm` channel on the [MacAdmins Slack Team](https://macadmins.herokuapp.com/), and of course, report any issues or feature requests on the github page.
