---
layout: post
title: "Deploying a Rails + Solr + Unicorn + Nginx application with Chef + Knife + Capistrano on Digital Ocean -- Part I"
date: 2013-05-22 14:27
comments: true
categories: 
---

The following is a merging of a few blog posts and gists, primarily [here](http://matteodepalo.github.io/blog/2013/03/07/how-i-migrated-from-heroku-to-digital-ocean-with-chef-and-capistrano/), [here](http://shawn.dahlen.me/blog/2013/03/06/automate-provisioning-of-secure-servers/), and [here](https://gist.github.com/czarneckid/4639793). I try and capture how to provision a server with Chef &amp; Knife and deploy a Rails application using Capistrano that uses Unicorn with Nginx, a small MySQL server, and a Solr index running in Jetty. Pretty much your standard [Blacklight](https://github.com/projectblacklight/blacklight) stack.

 Have a look and let me know what you think. Especially suggestions to make things better. 

Some things starting off:

* You can see the Chef code at [https://github.com/cfitz/catalog-kitchen/](https://github.com/cfitz/catalog-kitchen/) and the app code at [https://github.com/cfitz/blacklight_app.git](https://github.com/cfitz/blacklight_app.git) .
*   There's no shortage of hosting providers, and many of them have Knife plugins. EC2, Linode, EngineYard ... they're all great. I like [Digital Ocean](https://www.digitalocean.com/), so that's what I'm using here.
*   For this demo, the application is named "catalog" and will be hosted on catalog-dev.wmu.se.
*   I'm using [librarian](https://github.com/applicationsonline/librarian) for dependencies<span style="color: #333333; font-family: Helvetica, arial, freesans, clean, sans-serif;"><span style="font-size: 15px; line-height: 20px;">.</span></span>
*   This is all tailored to deploying a Blacklight application, which requires some specific Solr configuration. I try to make it obvious where that happens.
*   I'm still using RVM because I suck.

### Provision the server

 Before I can do anything, I have to install the knife-solo and digital ocean gems:

``` 
$ gem install knife-solo knife-digital_ocean
``` 

And now I make our Chef project (or "kitchen"). As I said, the project's name is "catalog"...

``` 
$ knife solo init --librarian catalog-kitchen
``` 

As [Shawn Dahlen states](http://shawn.dahlen.me/blog/2013/03/06/automate-provisioning-of-secure-servers/), even when running Chef solo it's a good idea to check your Chef project into a Git repository, which allows you to tag and branch your releases. So, I do that:

``` 
$ cd catalog-kitchen
$ git init
$ git add .
$ git commit -m "init commit"
$ git checkout -b dev
``` 
For provising our servers, we can do that either on Digital Ocean's web interface, or we can use Knife. To use knife, find your APIs keys in the Digital Ocean  dashboard and add them to a file named [.chef/knife.rb](https://github.com/cfitz/catalog-kitchen/blob/master/.chef/knife.rb).

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/.chef/knife.rb" ></script>

.. and I always add this to my .gitignore file.

``` 
$ echo .chef/ &gt;&gt; .gitignore
``` 

It's also a good idea to upload your public SSH key to your Digital Ocean account.

After you've done that, I can get a list of your keys by executing this in your project directory:

``` 
$ knife digital_ocean sshkey list
``` 

To get a sense of what your options are, you can use Knife to run of list of what Digital Ocean offers. There are various OS options, image sizes, and hosting locations. You can see what's available using some of these command:

``` 
$  knife digital_ocean region list # list hosting regions
$  knife digital_ocean size list  # list image sizes 
$  knife digital_ocean image list # list your images 
$  knife digital_ocean image list --global # list all the images
``` 

You can see the whole list of commands at [https://github.com/rmoriz/knife-digital_ocean](https://github.com/rmoriz/knife-digital_ocean).

So let's finally make our damn server already. I can do that with this command:

``` 
$ knife digital_ocean droplet create --server-name catalog-dev.wmu.se --image 2676 \
    --size 63 --location 2 --ssh-keys 11701
```

If you read about that page with list of commands, you've probably guessed that this creates a single core 1GB 30GB server, running Ubuntu 12.04 server in Amsterdam with my account's SSH keys preloaded.

After you run the create command, you'll be given a IP address (which you can use to setup your DNS) and get an email with your root password.

Now before I do anything else, log into the root account and change the password or better yet just turn off  password authentication for ssh access.

### Chef Up

First we need to make our cookbook:

``` 
$ knife cookbook create -o ./site-cookbooks catalog
``` 

This will scaffold out a file directory called ./site-cookbooks that we will use add our own recipes. But one of the great things about Chef is that there's lot of recipes you can find out there to do more of the work for you.
Take a look at the ./Cheffile

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/Cheffile" ></script>

This list out all the third part recipes we're going to need for this project. If you find a recipe you'd like to use, you can add it here.
Also have a look at site-cookbooks/catalog/attributes/default.rb.

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/attributes/default.rb"></script>

I guess now is a good time to go over user creation.

You're going to want more user accounts than just root, especially since as you see from the attributes/default.rb,  we're going to block off ssh root login. The easiest way to make user account is with data bags. Data bags are just json files of key/value attributes. They are located in the [./data_bags](https://github.com/cfitz/catalog-kitchen/tree/master/data_bags) folder of the project's root directory.

For this project, I am going to make two users : [Me](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/users/chrisfitzpatrick.json) and [rails](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/users/rails.json). There are located in [./data_bags/users ](https://github.com/cfitz/catalog-kitchen/tree/master/data_bags/users)

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/data_bags/users/chrisfitzpatrick.json" ></script>

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/data_bags/users/rails.json" ></script> 

The rails user will be the user that capistrano logs in to deploy the application. I do not want that user to be privileged. However, I want my user to be all-powerful.

To make the passwords, you can hash them out using mkpasswd -m sha-512 or (for OS X users) (for OS X users)  echo -n "a password" | openssl dgst -sha512. You can also add in your public ssh key here and your dot files if you're using Homesick (you should).  The users that are to be created are added to the default.users array in the attributes/default.rb file.

But what about all those other "accounts" I need to set up? My MySQL root user, my email SMTP account, blah blah blah. For this we're going to want to use encrypted databags. However, chef solo seems to have some issues with these, so we're going to need a little help.

Take a look at [this guy's gist ](https://gist.github.com/aaronjensen/4123044), which will give you a way to edit encrypted data bags using Chef solo. I copied  that gist in [directory called "libraries" ](https://github.com/cfitz/catalog-kitchen/blob/master/libraries/edit_bag.rb)in the project folder and  made a secret key like this:

``` 
$ openssl rand -base64 512 &gt; data_bag_key
$ echo data_bag_key &gt;&gt; .gitignore
``` 

So, the [ Percona recipe ](https://github.com/phlipper/chef-percona) requires an encrypted data bag to install MySQL, so I make a file called data_bags/passwords/mysql.json that initially just has  this:

``` json
 {
  "id": "mysql"
}
``` 
Now we can add our encrypted passwords like by running this from our project root directory:

``` 
$  ./libraries/edit_bag.rb passwords mysql
``` 

Your text editor should open to edit your mysql.json. Add "user": "password" values for users "root", "backup", and "replication". When you save the file, the script will encrypt your databag for you.

[Boosh](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/passwords/mysql.json).

It's a little akward, but it seems like the easiest way to go to encrypted data bags using the new version of Chef Solo.

I do the same for [system.json](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/passwords/system.json) (for mysql's log rotater), [rails.json](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/passwords/rails.json) ( for the app's database credentials ), and [gmail.json](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/passwords/gmail.json) (which is what I going to use as a dinky email relay) and [newrelic.json ](https://github.com/cfitz/catalog-kitchen/blob/master/data_bags/passwords/newrelic.json)(from system monitoring).

Now that we have most of our default settings and data bags in place, lets move on to setting up our recipes. First I run these commands:

``` 
$ librarian-chef install # this is like a bundle install
$ knife solo bootstrap root@catalog-dev.wmu.se
``` 

The first command will install all your third-party cookbooks to the cookbooks directory, just like a bundle install.

The second command will install a [json file in the node directory](https://github.com/cfitz/catalog-kitchen/blob/master/nodes/catalog-dev.wmu.se.json).  This json file sets what recipes you want to run (in the aptly named "run_list" value) as well as any node-specific attributes you want to define. Here you can add or remove any recipes you want from the list.

For this project, I'm running two main recipes, "requirements" and "default". 

Have a look at [site-cookbooks/recipes/requirements.rb](https://github.com/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/recipes/requirements.rb).

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/recipes/requirements.rb"></script>

Now look at the default recipe: 

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/recipes/default.rb"></script>

At the end of this recipe, I run two more site-cookbook recipes :catalog::solr-config and catalog::rails_application. 

First site-cookbooks/recipes/solr-config.rb :

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/recipes/solr-config.rb"></script> 

and site-cookbooks/recipes/rails_application (I have not idea why I went underscore on that one): 

<script src="http://gist-it.appspot.com/github/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/recipes/rails_application.rb"></script>

I'd recommend taking a hard look at the [nginx.erb](https://github.com/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/templates/default/nginx.erb) template. 
For this appication, I'm keeping a non-SSL path open to query Solr, so I can do some AJAX queries that will not require the processor overhead SSL requires. For everything else, it gets pushed to the SSL site. The /solr path also requires some basic auth, which we setup in the solr-config recipe. Of course, you can lock Solr down more if you chose and probably should if you are storing sensitive data. 

And with that, we're pretty much ready to run this thing. 

``` 
$ knife solo bootstrap catalog-dev.wmu.se
``` 

You should see all the recipes run and you can see if the web server and jetty is running by going to you host. 

In [the next post](/blog/2013/06/08/deploying-a-rails-plus-solr-plus-unicorn-plus-nginx-application-with-chef-plus-knife-plus-capistrano-on-digital-ocean-part-2/) I'll go over how to deploy you rails application with Capistrano.
