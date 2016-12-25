+++
date = "2016-07-31T22:45:44-04:00"
draft = false
title = "Nodeless Puppet for macOS" 

+++

When using puppet, it's recommended that you assign a unique [role](http://garylarizza.com/blog/2014/02/17/puppet-workflow-part-2/) to each [node](https://docs.puppet.com/puppet/latest/reference/lang_node_definitions.html). The role could be `webserver`, `admin_workstation`, `lab_mac` and so on. There are several ways to do this, the most simple one is by having a list of nodes in `manifests/site.pp`, and there are sevaral abstractions, like Foreman, or the Puppet Enterprise Dashboard, which allow you to assing roles to nodes through a web interface. If this sounds vaguely familiar to you, it should -- munki recommends the same strategy for manifests. We create a unique manifest for each hostname/serial number and use included manifests to add common applications.

<!--more-->

As both a munki or puppet user, I found the need to define the same data in two locations (unique munki manifest and puppet node definition) somewhat redundant, and potentially error prone. I find it preferable to have a [single source of truth](https://en.wikipedia.org/wiki/Single_source_of_truth). What if instead of defining this inventory data in two different software repositories, we let one tool define the data in the other. One way to achieve this, is to allow puppet to manage the manifests file in your munki repo. It can work, but I chose to do the opposite -- assigning roles in puppet, based on some metadata present on the Mac.

I originally came accross this idea from [Jordan Sissel's](https://twitter.com/jordansissel) puppet-examples [repository](puppet-examples/nodeless-puppet) several years ago and have adopted it for my environment - mac workstations.

# The default node

> The name default (without quotes) is a special value for node names. If no node statement matching a given node can be found, the default node will be used. [See Behavior below](https://docs.puppet.com/puppet/latest/reference/lang_node_definitions.html#the-default-node).

Just like munki, puppet allows specifying a default node definition which will be used in case it fails to find a specific configuration. We can take advantage of this configuration to apply a dynamic classification to nodes. 

```
# site.pp
node default {

    # switch on the role fact
    case $::role {
      'admin_workstation': { include role::admin_workstation } 
      'labmac':            { include role::lab_mac  }          
      'student':           { include role::student_mac  }      
      default:             { include role::generic }           
    }
}

# override a specific node 
# will be applied instead of the default node
node snowflake.mycompany.com {
    include role::special_snowflake
}
```

Now we have a very short `site.pp` which applies the correct roles to the macs in your organization. We also have the ability to override the default classification with a specific node hostname.
At this point, you're probably asking - how does the `role` fact end up on the Mac? The answer is - simple textfile with `role=mac_role` key/value pair.

# Imaging Macs with custom metadata
There are multiple ways to automate this next step, especially if you have a good inventory system. I'll show you one where I assume you have nothing but a munki repo or imagr/deploystudio server.

[Custom facter facts](https://docs.puppet.com/facter/3.1/custom_facts.html#external-facts) can simply be a plain text, json or yaml file in `/etc/facter/facts.d/`
My metadata lives in `/etc/facter/facts.d/hardware_tag.txt` and looks like this:
```
role=labmac
building=nh
room=101
dualboot=true
```

Calling `sudo /opt/puppetlabs/bin/facter -p role` results in `labmac`.
My munki manifest for `nh-101` includes a specific package -`pupept-tag-nh-101.pkg` which places the `hardware_tag.txt` file into the correct location.

In the interest of space, I created an example Makefile for this package in a separate [gist](https://gist.github.com/groob/d16aa82d01a47e2714e6f5bf75067ec3).

If you're interested in managing workstations with puppet, join us on the [MacAdmins](https://macadmins.herokuapp.com/) slack in #puppet or ping me on [twitter](https://twitter.com/wikiwalk).
