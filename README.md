# Frycook

Frycook is a quick and dirty alternative to Chef.  When a giant kitchen full of equipment and a full staff headed by a master chef are *WAY* more than you need, a frycook will do the trick nicely.  It'as also nice when you'd rather do things in Python than have to learn Ruby or some other proprietary DSL.

Frycook is a thin layer on top of fabric, cuisine, and mako.  Frycook consists of the frycook package you install via pip and a set of recipes and metadata files you generate to describe your environment and how to build it.  I'll call this set of recipes and metadata a _globule_ in this document.  There is a sample globule in the sample directory that handles a base server setup and web domain setup for a simple web server, based on Ubuntu 12.04.

The ideas embodied in frycook originally came from managing several hundred servers for a social gaming company, and then were augmented by time spent poring over the chef documentation.

The best way to learn frycook is to look through the code in the frycook module and in the sample globule that's included with this repo.

# Story

When I build a server, I prefer to write a program to build the server instead of just logging in to the server and configuring things by hand.  This way, when the server eventually crashes (or I need to build a second copy to scale things up), I have a program that can quickly build a new copy of that server for me.  Later, when I need to change the configuration on the server, I just update the setup program and run that again.  That way my setup program is always up to date.  

The first way I did this was by writing bash scripts that made liberal use of ssh to connect to the server and run commands on it.  This works pretty good.  Eventually, though, you end up with a huge pile of bash code that gets unwieldy pretty quickly.  Combine that with the fact that I find bash to be a little opaque and I finally went looking for a better solution.  I checked out all of the popular solutions (Chef, Puppet, etc...) and didn't find one that felt like a good fit for me.  They all have lots of complex infrastructure to setup and require learning things like Ruby or proprietary DSLs, none of which caught my fancy.

I spend most of my time writing code in Python, so I decided that a Python solution would be most palatable.  This inevitably led to fabric, which is the go-to solution for handling ssh, and cuisine, which adds a bunch of neat commands on top of fabric to handle lots of common administrative tasks on remote servers.  Still, though, these weren't enough.  I wanted to add in the ability to track *all* the metadata for my servers in one central place (but not in a database server), and to be able to generate configuration files from templates (I like Mako for this).

Finally, I decided to bite the bullet and roll my own system.  I figured it would be fun and give me a chance to get up close and personal with fabric and cuisine and learn a bit about how they work internally.  Also, this way I would have complete control of my build system and understand it intimately.  Sometimes I'm kind of a control freak that way.

Although I didn't choose to use chef or puppet, I really liked some of their concepts and I decided to make them part of the foundation for my system.  Chef has the nice concept of recipes for individual subsystems and cookbooks for building multiple subsystems into complete systems.  Chef also places a lot of value on idempotency, making sure that you can run the same setup script on a server multiple times and not cause yourself problems in the process.

# frycooker

This is the job runner.  It takes as arguments a recipe or cookbook to run and a list of computers and groups to push the packages to.  Groups are expanded to computers and the contents are pushed to each one.

# Globules

A globule contains all the files and metadata that frycook will use to help create and manage your environment.

## setup directory

The setup directory contains all the recipes, cookbooks, and metadata.

### environment.json file

Here you keep all the metadata about the users, computers, and groups for your installation.

### runner.sh file

This is just a wrapper around frycooker that sets the PYTHONPATH correctly before running frycooker.

### recipes directory

A recipe describes how to install an individual package, such as postfix or nginx.  Frycook expects all your recipes to be in a module/package called 'recipes' that is accessible via the PYTHONPATH environment variable.  This module should export a list of recipe name and class tuples for use by the frycooker program.

### cookbooks directory

A cookbook is just a list of recipes to install.  You can add other logic to it, but basically it's just a list of recipes.  You need to put all your cookbooks in a module/package called 'cookbooks' 
that's in the PYTHONPATH environment variable.

### settings.json file

There are a few configuration settings for frycook, and they're set here.

## packages directory

This directory keeps copies of all the files for each recipe.  One of the major things a recipe does is copy config files to the server, and here's where they live, one directory per package.  Plain files are copied verbatim, while files with a .tmplt extension are treated as mako templates and processed before being copied to the server.

# recipes

A recipe is a python class that you subclass and override certain methods to hook in to different parts of the process that applies recipes.  The two main parts are apply and cleanup.

## apply

This is where you apply a recipe to a server.  There are three class methods that get called during the apply process. Generally you'll just override the apply method.  If you override pre_apply_checks, remember to call the base class method.

pre_apply_checks -> apply -> post_apply_cleanup

## cleanup

This is where you cleanup old recipe configurations from a server.  An example is when I changed the home directory for my web server.  I first wrote a cleanup that cleaned-up the old configuration, then an apply to apply the new configuration.  That way you can always run the apply in the future when building new machines and don't need the cleanup logic since the new machine never had the old configuration that had to get cleaned-up.

# cookbooks

# templates

# idempotency

One thing to keep in mind when creating recipes and cookbooks is idempotency.  By keeping idempotency in mind in general you can create recipes that you can run again and again to push out minor changes to a package.  This way your recipes become the only way that you modify your servers and can be a single chokepoint that you can monitor to make sure things happen properly.

Lots of the cuisine functions you'll use have an "ensure" version that first checks to see if a condition is true before applying it, such as checking if a package is installed before trying to install it.  This is nice when things could cause undesired configuration changes or expensive operations that you don't want to happen every time.  These functions are a huge aid in writing idempotent recipes and cookbooks.