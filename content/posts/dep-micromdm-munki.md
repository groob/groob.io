+++
date = "2016-04-24T17:08:10-04:00"
title = "Munkiing around with DEP"

+++

In my [last post](http://groob.io/posts/mdm-experiments/) from November I wrote an introduction to Apple's MDM Protocol spec. Apple has shown an innovative approach to enterprise deployment with it's DEP service and MDM protocol. Apple's solution allows for a more flexible deployment for administrators while giving users more control over their devices. Most enterprises already have robust solutions to manage devices in their organizations - especially laptops and desktops. We use Imagr, Munki and Puppet internally to manage our users' machines.  
Unfortunately there are not many options for integrating existing solutions with Apple's new services. MDM vendors offer costly solutions and expect you to adopt their whole platform, no matter how poorly it fits in with your environment. To make matter worse, most MDM vendor's out there don't implement the parts of MDM spec for OS X, instead focusing mostly on iOS. 


I've been looking for an MDM solution that would integrate with the [DEP service](http://www.apple.com/business/dep/) and allow an administrator to assign devices to a group/organizational unit. When that device would arrive in the hands of the end user, the MDM server would bootstrap Munki and Puppet. At that point, our existing services would configure the device as needed.
Frustrated by the MDM vendor landscape out there, and inspired by the [work of others](http://enterprisemac.bruienne.com/2015/11/17/installing-os-x-pkgs-using-an-mdm-service/), I decided to write my own. 

I began building an open source MDM server that should fit my needs and hopefully others' too. Today, I want to show you what I got so far. It's not yet ready for release but the source is out there if you want to take a look.

# micromdm

[micromdm](https://github.com/micromdm/micromdm) is an http server written in Go which implements Apple's MDM Protocal specification. It is fully API driven and can be integrated with other existing services in our enterprise. The server itself is a single binary that can run on Linux/Windows and OS X but it's also designed in a way that allow splitting individual components into their own services. 

The first release will be focused on deploying existing Mac management tools onto new devices. I made a (not so) short video to show how a device enrolled with DEP is set up with Munki the first time. For this demo, I've split up a process which should be automated in a workflow and logged all requests/responses between devices and server to `stdout` so that I can explain how the pieces work.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Ugsl7CMlqi4" frameborder="0" allowfullscreen></iframe>

# InstallApplication 

If you survived until the end of the video you'll notice that Apple's current agent does not allow the `InstallApplication` command to be triggered at the loginwindow. Using an MDM we can only install/begin installing applications once a user logs in. This limitation makes DEP less useful in some scenarios but we can mix in imaging thin images in those scenarios. 
Currently, even in lab environment we have an image with just following ->

* munkitools.pkg
* kickstart.pkg to [bootstrap](https://github.com/munki/munki/wiki/Bootstrapping-With-Munki) munki.
* SuppressSetupAssistant
* local_admin_account.pkg

With DEP, we can create an image with just the kickstart pkg and then leverage the SetupAssistant/DEP for local admin account and munkitools. While the benefits to doing that are not immediately obvious to doing that, it's nice to have the option.

# Configuration

One of the advantages of DEP is that it creates an inventory of all new devices purchased from Apple by our organization. The DEP API allows us to automate the creation of Munki manifests at the same time as our devices are enrolled into MDM.

Another interesting bit to explore is authentication and user account creation during DEP setup. When the device first contacts the server, the server can reply back with a `401` status and an `WWW-Authenticate: Digest` header, prompting the user for credentials.
Here we can use the credentials provided by the user to provision a local user account, verify the account against LDAP or do a number of other things.

The MDM protocol is it's in early stages and will add more features, but I'm also hoping that an open source implementation will allow more people in the community to jump in and find ways to extend the protocol.
