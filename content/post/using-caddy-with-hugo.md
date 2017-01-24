+++
date        = "2017-01-24"
title       = "Running a secure blog with Caddy and Hugo"
description = "Using the Caddy web server to run a Hugo site with HTTPS"
tags        = [ "Caddy", "Hugo", "Docker", "hosting", "secure", "https", "Let's Encrypt" ]
topics      = [ "Development" ]
+++

It's about time that I got my feet wet in this bloggin' world of ours. Great! But, oh wait, which blog software do I use? And a secure HTTPS site sounds good, but how on earth do I do that?

## Show me the technologies!

As per usual in todays world, we're going to be bringing a collection of technologies together to achieve glory.

* [Docker](https://www.docker.com/) - we'll be using Docker to run all the things and simplyfy life as we know
* [Caddy](https://caddyserver.com/) - this great little web server will host our static site and provide glorious HTTPS using Let's Encrypt certificates
* [Hugo](https://gohugo.io/) - a great little static site generator

## Creating a Hugo site

You can download a statically compiled executable for most operating systems, but, I dislike adding too many tools to my path. So, instead, let's just run this thing in Docker!

    $ mkdir ~/my-hugo-site
    $ cd ~/my-hugo-site
    $ docker run -it --rm -v $(pwd):/usr/share/blog publysher/hugo hugo new site .

Here we are creating the directory we want to put the site in to and then running hugo (from inside the official Docker container) to create a new site.

Want to see what the site is going to look like? Great! Let's server it up with the hugo server.

    $ docker run -it --rm -v $(pwd):/usr/share/blog publysher/hugo

I won't go in to how to add all of your words of wisdom in to this flashy new site, but feel free to take a detour and check out the [official documentation](https://gohugo.io/overview/introduction/).

## Let's get Caddy-ing!

Are you ready to run this magical new site of yours in a fancy web server! Great! Let's get to it.

    $ docker run -d -v $(pwd)/public:/srv -p 2015:2015 abiosoft/caddy

This will spin up a Caddy server on port 2015. Check it out, you should now see your site running on [http://localhost:2015](http://localhost:2015)!

## Tired of typing all those characters? docker-compose to the rescue!

I don't know about you, but I sure hate having to remember and type out those long Docker commands all the time. What if I told you we could encode all of that in a pretty little YAML file and then run it with `docker-compose up`!

First, let's stop any running docker containers.

> WARNING: This will stop ALL running Docker containers. If you have other containers running that you want to keep running, don't use this! Instead, shut down your Hugo + Caddy containers manually

    $ docker stop $(docker ps -aq)
    $ docker rm $(docker ps -aq)

```yaml
hugo:
    image: publysher/hugo
    command: hugo
    volumes:
        - .:/usr/share/blog

caddy:
    image: abiosoft/caddy
    volumes:
        - ./public:/srv
    ports:
        - 80:2015
```

    $ docker-compose up

## Let's get hosting!

So that's everything working locally. Great! But now it's time to get serious.

Since we're running everything in Docker containers, we have a vast array of options for where we can host the site. I'm a big fan of [Digital Ocean](https://www.digitalocean.com/), so that's what I'm going to use.

> Use my [referral code](https://m.do.co/c/dcb793d3dc9e) to sign up to Digital Ocean and get $10 credit right off the bat!

### Create a new droplet

So, I went ahead and created a droplet using:

* The "Ubuntu Docker 1.12.6 on 16.04" One-click App
* $5/mo droplet (512 MB / 1 CPU / 20 GB SSD / 1000 GB transfer)

### Log in, clone the blog and spin it up

Now that a droplet is up and running, we can log in and get things running. This is an almost identical process to what we did locally, but, well, it's on the line.

    $ ssh my-droplet-server-ip
    $ git clone https://github.com/[Your username]/hugo-blog.git
    $ cd hugo-blog
    $ docker-compose up -d

And would you look at that, you're blog is now there for the entire world to peruse!

### Point your domain at your droplet

An IP address is all fine and dandy, but it's not the easiest thing to remember. So, put your domain or subdomain in front of it!

So, go ahead and create an A record with your desired hostname (e.g. blog.garjon.com) that points at your droplets IP address. I also went and added a couple of CNAME records so that `www.garjon.com` and `garjon.com` will resolve to `blog.garjon.com`. Go out and read up on DNS records if you need more information (and feel free to correct me if you disagree with how I've set mine up!).

#### Example of what my DNS records looked like
![DNS Records](/img/using-caddy-with-hugo_dns-records.png)