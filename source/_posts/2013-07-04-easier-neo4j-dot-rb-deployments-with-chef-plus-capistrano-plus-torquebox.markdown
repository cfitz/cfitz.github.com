---
layout: post
title: "Easier Neo4j.rb deployments with Chef + Capistrano + Torquebox"
date: 2013-07-04 11:58
comments: true
categories: neo4j chef rails jruby
---

This is part follow-up to my previous two posts and part a response to some questions I've seen on the [Neo4j.rb group](https://groups.google.com/forum/#!forum/neo4jrb) about deployment. A big drawback of using Neo4j.rb is that it cannot be deployed to Heroku, since it uses an embedded database and not Neo4j's REST API. However, setting up a server to host your application isn't as hard as you might think and you'll get much better performance.  The following is how I do it with Chef and Capistrano.

Before we start: 

* I use [Digital Ocean](https://www.digitalocean.com/) for VPS hosting, but Linode and EC2 are also great. 
* I used Ubuntu 12.04 64-bit on this. I don't think there's anything that would break this on other Linux distros, but I'm not sure...
* This uses [Torquebox](http://torquebox.org/) for an application server because it's awesome. 
* In order to keep it simple, this demo also uses Nginx as a reverse proxy. If you want mod_cluster support, you'll have to use Apache2. Have a look at [this](https://github.com/torquebox/chef-cookbooks/tree/master/mod_cluster). 


### Part I: Server by Chef

A copy of the Chef recipe is [here](https://github.com/cfitz/neo4j-torquebox-kitchen). I'll be using knife-solo to keep it simple.
Let's clone this:

```
$ git clone https://github.com/cfitz/neo4j-torquebox-kitchen
$ cd neo4j-torquebox-kitchen
$ bundle 
```

First, edit the nodes/neo4j-example.json file to change the "server_name" and "application_name" to match the domain name of your server. Or you can just us IP address if you don't have a name. Just note that for the rest of this blog, I'm going to referrer to the application as "neo4j-demo-app.com". 

The next thing you're going to need to change is the data_bags/users/torquebox.json file. This is the configuration for the torquebox user that Chef will create on the server. You'll need to make a new encrypted password (this one in the demo code is just "password"). To do that, run:

```
$ openssl passwd -1 "theplaintextpassword"
or
mkpasswd -m sha-512
```

Copy and paste the output into the JSON's password key. You'll also want to add your ssh_key to this file. And you can delete the homesick_castles entry if you want, as it will install a dotfile in your user's home directory (cool, but not essential. If you want to use this, I'd suggest forking it and editing it to your liking). You can also add any other users that you want by simply replicating this JSON file, changing the values, and adding the username to the default.users array in site-cookbooks/neo4j-example/attributes/default.rb . 

Once you have your node and users configured, we can run this:

```
$ bundle exec librarian-chef install # this is like a bundle command to update our recipes.
$ bundle exec knife solo bootstrap root@neo4j-demo-app.com nodes/neo4j-example.json 
```

Change the name of the server to match your server name or IP.

This might take awhile, since we're having to install a few packages. Among other things, this will: 

* Install a user named torquebox.
* Download and install torquebox 2.3.2 into /opt/torquebox 
* Install and configure Nginx as a reverse proxy
* Locks down ssh to not allow password authentication and not allow root user access.
* Sets a minimal iptables configuration to only allow ports 80, 443,  and 22. 
* Install fail2ban, vim, tmux, git, and some other packages, which I always like to have. 

If you go to your servers root URL, you should get a "502 Bad Gateway" from Nginx. This is good news for now...so let's deploy an app!


### Part II: App by Rails

I made this [demo application](https://github.com/cfitz/neo4j-demo-app) as a reference by following the [Complete Example on the Neo4j.rb wiki](https://github.com/andreasronge/neo4j/wiki/Neo4j%3A%3Aha-cluster) with a couple of gems added. These are: 

```
gem "neo4j", "2.3.0"
gem 'neo4j-community'
gem 'neo4j-advanced'
gem 'neo4j-enterprise'
gem "torquebox-server"
gem "torquebox"
gem "torquebox-rake-support"
gem 'torquebox-capistrano-support', :group => :development
gem 'capistrano'
```

Add those gems to your application, bundle it, and you'll be ready to run it. But let's take a brief detour...

#### Intermission: Run Your App Locally with Torquebox

The torquebox-server gem has an embedded version of Torquebox that you can use to run your application. To do that, run this command: 

```
$ bundle exec torquebox deploy # this "deploys" your application by generating a torquebox knobfile.
$ bundle exec torquebox run 
```

Go to [http://localhost:8080/users](http://localhost:8080) and you should see you're application. 

Back to deploying it.

### Part III: Deployment by Capistrano

First, the easiest way to deploy with Capistrano is to check your application into version control, like Github.

```
$ git init
$ git add .
$ git commit -m "initial commit"
$ git remote add origin git@github.com:cfitz/neo4j-demo-app.git
$ git push origin master
```
 

To setup Capistrano, run this command from your Rails root:

```
$ bundle exec capify .
```

This add a few files to your rails application. First is Capfile. Make sure to comment our "load 'deploy/assets". 

Now look at the config/deploy.rb file (see the inline comments on how this all works): 
<script src="http://gist-it.appspot.com/github/cfitz/neo4j-demo-app/blob/master/config/deploy.rb"></script>


You'll have to change a few values to match your server name and application name. When you're ready to deploy, run:

```
$ bundle exec cap deploy:setup # this creates the folder structure for your application. Only do it once on your initial run
$ bundle exec cap deploy
```


This will:

* Check your application out in Git and deploy it to /opt/apps/neo4j-demo-app.com
* Compile your assets locally and scp them to the server.
* Deploy a torquebox knobfile and start your application.


You should not have to start torquebox on deploys, but if you want to control your server, you can do that with Rake: 

```
$  bundle exec cap deploy:restart 
$  bundle exec cap deploy:start
$  bundle exec cap deploy:stop
```


Now you should go to http://neo4j-demo-app.com/users (or whatever your URL is) and see your application. 
Any time you want to update your code, simply check it into git and run cap deploy. 

This might be more work than Heroku, but you'll get much better performance off the bat and it will not cost you an arm and a leg when you want to scale out. 

Suggestions / Questions always welcome. 






