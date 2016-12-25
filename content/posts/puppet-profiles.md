+++
date = "2016-07-31T15:00:22-04:00"
draft = false
title = "Dynamic Configuration Profiles with Puppet"

+++

These days you can get a lot of configuration of OS X done by applying a [profile](https://www.youtube.com/watch?v=gTw3U-vxung) to manage the setting you want. There are a myriad of ways to deploy a configuration profile - using MDM, Munki, Casper, email, etc. One thing that gets tedious with profiles is managing the same setting, accross multiple groups. If you have to set a different Desktop picture for five different groups, that means maintaining five different profiles. This is the kind of problem that configuration management tools were created to solve, and today I'll show you how to easily manage OS X settings with [puppet](https://puppet.com/).

# Creating a template

What sets puppet appart from the other ways of deploying profiles, is that puppet is able to compile the profile from a template, substituting the keys from a database. I'll start with an example, and explain how it works later in the blog post.
First, let's take a sample profile.
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>PayloadContent</key>
	<array>
		<dict>
			<key>PayloadDisplayName</key>
			<string>Desktop</string>
			<key>PayloadEnabled</key>
			<true/>
			<key>PayloadIdentifier</key>
			<string>...identifier</string>
			<key>PayloadType</key>
			<string>com.apple.desktop</string>
			<key>PayloadUUID</key>
			<string>...uuid</string>
			<key>PayloadVersion</key>
			<integer>1</integer>
			<key>locked</key>
			<true/>
			<key>override-picture-path</key>
            <string>/Library/Desktop Pictures/El Capitan.jpg</string>
		</dict>
	</array>
	<key>PayloadDescription</key>
	<string>Manage Desktop background</string>
    ...some more keys here...
</dict>
</plist>
```

In this case, we want to manage the Desktop Picture path, which corresponds to the `override-picture-path` key. To turn this profile into a template, we'll replace the path, with a variable we can manage in Puppet's database - Hiera.
```
<key>override-picture-path</key>
<string><%= @desktop_background_path %></string>
```

Now we can substitute `desktop_background_path` with the actual profile in our manifest.

# Installing the profile with Puppet

Puppet has a module for installing profiles -- [keeleysam/puppet-mac_profiles_handler](https://github.com/keeleysam/puppet-mac_profiles_handler) which is maintained by Samuel Keeley, and occasinaly other Mac Admins. To install the profile, we'll make a manifest. 


```
# manages the desktop and loginwindow background on macOS 
class profile::mcx_profile::desktop_background (
  $ensure = 'present',

# The desktop_background_path variable is referenced in the ERB template and 
# overwritten in hieradata for each role
  $desktop_background_path = '/Library/Desktop Pictures/El Capitan.jpg'
) {
# here we reference the profile by its identifier
  mac_profiles_handler::manage { 'com.mycompany.desktop_background':
    ensure      => $ensure,
    file_source => template('profile/mcx_profile/com.mycompany.desktop_background.erb'),
    type        => 'template',
  }
}
```

The `mac_profiles_handler` resource will manage the profile, substituting the variable with the actual path, before installing the profile.  We can change `ensure` from `present` to `absent` to remove it.


Now, we can override this setting for different groups inside our hieradata.
```
---
profile::mcx_profile::desktop_background::desktop_background_path: "/Library/Desktop Pictures/MyCompany/Desktop.jpg"
```

# Additional resources

If you're not using puppet yet, you can learn more about getting started from Graham Gilberts great [talk on puppet](https://www.youtube.com/watch?v=Z2quMhdgILo) from MacADUK 2016.

I also recommend reading Garry Larizza's blog on [Puppet workflows](http://garylarizza.com/blog/archives/).

While working on this blog post, I created a new module [mac_profile](https://github.com/groob/puppet-mac_profile) built on top of the mac_profiles_handler which includes some common profiles. You can either use it as an example
or install it for your puppet repo. If you'd like to extend the module, feel free to submit a PR for adding more profile types.

