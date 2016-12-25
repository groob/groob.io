+++
date = "2015-05-16T16:56:41-04:00"
title = "Intro to creating OS X Packages"

+++

For years I've used tools like [The Luggage](https://github.com/unixorn/luggage) and [Whitebox Packages](http://s.sudre.free.fr/Software/Packages/about.html) to package scripts and deploy files to end user machines. The Luggage is cumbersome, and I'm a bit averse to GUI tools that I can't easily version control and automate. I've expressed this frustration in ##osx-server a couple weeks ago, and Tim Sutton replied with 
```
tvsutton > mkdir "root"; <put stuff in root>; pkgbuild --root root --identifier foo.pkg --version 1.2.3 my-output.pkg
```
As it turns out, this oneliner is about as difficult it gets to create a new package. Let's take a look at a real example:

Bootstrap munki by placing the file `.com.googlecode.munki.checkandinstallatstartup` into `/Users/Shared`.

```
# create the root folder for our package. It can be named anything.
$ mkdir pkgroot 
# create the Users/Shared directory inside our package root.
$ mkdir -p pkgroot/Users/Shared 
# create the empty file
$ touch pkgroot/Users/Shared/.com.googlecode.munki.checkandinstallatstartup 
# build our package
$ pkgbuild --root pkgroot --identifier org.myorg.munkikickstart --version 1.0.0 ~/Desktop/munki_kickstart.pkg
```
And that's it! We have a package that we can deploy.

What if we wanted to add a pre or postinstall script to our package? Well that's simple too. 
First, we'll create a folder called `scripts` and place our scripts there. Then, we just add --scripts /path/to/scripts/folder when calling pkgbuild.

[Here](https://github.com/whitby/mac-scripts/blob/master/ssltrust/Makefile#L9)'s another example of a package that I use to add a certificate to my keychain. It uses a postinstall script.


I hope that's enough to get you started as well.
