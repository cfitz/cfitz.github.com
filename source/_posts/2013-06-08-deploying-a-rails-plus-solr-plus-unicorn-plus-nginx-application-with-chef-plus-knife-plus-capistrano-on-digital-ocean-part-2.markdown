---
layout: post
title: "Deploying a Rails + Solr + Unicorn + Nginx application with Chef + Knife + Capistrano on Digital Ocean -- Part II"
date: 2013-06-08 22:33
comments: true
categories: 
---

Alright, so as the title suggests, this is the second part of an explanation of how I am using Chef and Capistrano to deploy Rails applications. Part 1 can be found [here](/blog/2013/05/22/deploying-a-rails-plus-solr-plus-unicorn-plus-nginx-application-with-chef-plus-knife-plus-capistrano-on-digital-ocean-1/). 

Part 1 was all about using Chef to provision and configure the server. This has all been setup to show how to deploy a [Blacklight](https://github.com/projectblacklight/blacklight) application, which uses Solr for indexing, but I hope it's easy to see how you can tailor this to work with your application's needs. 

This part is all about using Capistrano to deploy the application to the server. This will probably be much easier to follow along, as it's not really that complicated. I do a couple of tricky things to have zero downtime deploys and local asset compilation, but I try to make it really obvious where I'm being essentially cute and when I'm being unessentially cute.

First, this all assumes you have a Rails application. You can set one up using the [Blacklight quickstart](https://github.com/projectblacklight/blacklight/wiki/Quickstart) or whatever you want, but you need to have an app. For your referencing pleasure, you can look at [this really simple Blacklight application](https://github.com/cfitz/blacklight_app) which I setup for this blog. And for Capistrno, you'll need to check this application into version control. 


Now, remember in Part I, I set up the [rails_application recipe](https://github.com/cfitz/catalog-kitchen/blob/master/site-cookbooks/catalog/recipes/rails_application.rb) to deploy a database.yml to the server with the database information already configured. So we don't need to really do anything with the database.yml.

### Unicorn

For production, I like to use Unicorn with Nginx serving as a reverse proxy. To configure Unicorn, we need to add a config/unicorn.rb file: 


<script src="http://gist-it.appspot.com/github/cfitz/blacklight_app/blob/master/config/unicorn.rb"></script>


and be sure to check this into version control...

### Setup Capistrano

Take a look at the [Gemfile](https://github.com/cfitz/blacklight_app/blob/master/Gemfile). You will need to add the capistrano, capistrano-ext, rvm-capistrano, capistrano-unicorn, unicorn gems and bundle that.

Now we need to setup Capistrano for the project:

``` 
$ bundle exec capify .
```

This will create a file at Capfile and a ./config/deploy.rb file. For this project, we can leave the Capfile as it is, since we're going to do asset compiling locally then scp the results to the server. But, if we wanted to make the server compile our assets, we can change that in the Capfile. 

I usually like to use the multi-stage feature of Capistrano, since I usually have a staging and production environment in play. To do this, make a config/deploy directory. To configure your staging server, you make a file in this directory call staging.rb

``` ruby staging.rb
set :server_ip, "catalog-dev.wmu.se"
server server_ip, :app, :web, :primary => true
set :rails_env, 'production'
set :branch, 'master'
```

All the action takes place in the ./config/deploy.rb file: 

<script src="http://gist-it.appspot.com/github/cfitz/blacklight_app/blob/master/config/deploy.rb"></script>


Once this is in place, we can deploy our application: 

```
$ cap deploy:setup
$ cap deploy:migrate
$ cap deploy COMPILE_ASSETS=true
```

The COMPILE_ASSETS option forces Capistrano to compile the assets. The default action will only compile if there have been changes made to your assets. On the  initial run, make sure you set the COMPILE_ASSETS, since the default will not compile on the first run since it will not find any changes in your history. 
