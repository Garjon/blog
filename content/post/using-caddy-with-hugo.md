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
* [Let's Encrypt](https://letsencrypt.org/) - a free, automated, and open Certificate Authority

## Creating a Hugo site

You can download a statically compiled executable for most operating systems, but, I dislike adding too many tools to my path. So instead, let's just run this thing in Docker!

    $ mkdir ~/my-hugo-site
    $ cd ~/my-hugo-site
    $ docker run -it --rm -v $(pwd):/usr/share/blog publysher/hugo hugo new site .

Here we are creating the directory we want to put the site in to and then running Hugo (from inside the official Docker container) to create a new site.

Want to see what the site is going to look like? Great! Let's serve it up with the Hugo server.

    $ docker run --rm -p 1313:1313 -v $(pwd):/usr/share/blog publysher/hugo

I won't go in to how to add all of your words of wisdom in to this flashy new site, but feel free to take a detour and check out the [official documentation](https://gohugo.io/overview/introduction/).

## Let's get Caddy-ing!

Caddy is a small web server that is very easy to configure and run.

Let's take a look at how easy it is to spin up a server with the default settings:

    $ docker run --rm -v $(pwd)/public:/srv -p 2015:2015 abiosoft/caddy

This will spin up a Caddy server on port 2015. Check it out, you should now see your site running on [http://localhost:2015](http://localhost:2015)!

But, default settings are not exactly what we're after here, so let's take a look at configuring Caddy.

Caddy is configured using a `Caddyfile`. You can read up more about the [Caddyfile on the Caddy website](https://caddyserver.com/docs/caddyfile).

Here, I'm creating a file called `Caddyfile-prod` (i.e. my production Caddyfile). Inside it, I've defined a HTTPS redirect as well as the blog address I want it to server up.

`Caddyfile-prod`
```
# Permanent redirect to HTTPS
0.0.0.0:80 {
  log stdout
  errors stderr
  redir https://blog.garjon.com{uri} 301
}

https://blog.garjon.com {
  log stdout
  errors stderr
  tls gareth.jones@garjon.com
  root /srv
}
```

I also created a separate `Caddyfile-dev` to make developing off the line a little easier.

`Caddyfile-dev`
```
localhost:80 {
  log stdout
  errors stderr
  tls off
  root /srv
}
```

## Tired of typing all those characters? docker-compose to the rescue!

I don't know about you, but I sure hate having to remember and type out those long Docker commands all the time. What if I told you we could encode all of that in a pretty little YAML file and then run it with `docker-compose up`!

First, let's stop any running docker containers.

> WARNING: This will stop ALL running Docker containers. If you have other containers running that you want to keep running, don't use this! Instead, shut down your Hugo + Caddy containers manually

    $ docker stop $(docker ps -aq)
    $ docker rm $(docker ps -aq)

A docker-compose file is a YAML file that describes the different Docker containers you want to run. You can then use the `docker-compose up` command to spin up all containers defined within that file.

I find it useful to maintain two versions of the docker-compose files, one for production and another for development. So, I went ahead and created a directory called `docker-compose/` and dropped put two files in it, `prod.yml` and `dev.yml`.

`prod.yml`
```yaml
hugo:
  image: publysher/hugo
  command: hugo
  volumes:
    - ..:/usr/share/blog

caddy:
  image: abiosoft/caddy
  restart: always
  ports:
    - 80:80
    - 443:443
  volumes:
    - ../Caddyfile-prod:/etc/Caddyfile
    - ../caddy:/root/.caddy
    - ../public:/srv
```

`dev.yml`
```yaml
hugo:
  image: publysher/hugo
  command: hugo
  volumes:
    - ..:/usr/share/blog

caddy:
  image: abiosoft/caddy
  restart: always
  ports:
    - 80:80
  volumes:
    - ../Caddyfile-dev:/etc/Caddyfile
    - ../public:/srv
```

With those files in place, we can now spin things up!

    $ docker-compose up -f docker-compose/dev.yml

Similarly, if we want to run up the production instances.

    $ docker-compose up -f docker-compose/prod.yml

## Version control time!

If you haven't already, created a new (or clone an existing) Git repository to hold all these wonderful files. The has the obvious benefit of versioning all your changes, but will also help getting this site online for you to host.

## Let's get hosting!

So that's everything working locally. Amazing! But now it's time to get serious, well, fun serious.

Since we're running everything in Docker containers, we have a vast array of options for where we can host the site. I'm a big fan of [Digital Ocean](https://www.digitalocean.com/), so that's what I'm going to use.

> Use my [referral code](https://m.do.co/c/dcb793d3dc9e) to sign up to Digital Ocean and get $10 credit right off the bat!

### Create a new droplet

So, I went ahead and created a droplet using:

* The "Ubuntu Docker 1.12.6 on 16.04" One-click App
* $5/mo droplet (512 MB / 1 CPU / 20 GB SSD / 1000 GB transfer)

### Log in, clone the blog and spin it up

Now that a droplet is up and running, we can log in and get things running. This is an almost identical process to what we did locally, but, well, it's on the line.

> This example shows me pulling down my blogs git repository. Clone your own one for maximum awesomeness!

    $ ssh my-droplet-server-ip
    $ git clone https://github.com/Garjon/blog.git
    $ cd blog
    $ docker-compose up -f docker-compose/prod.yml -d

If you point your browser at the droplet's IP address, you should see your magnificent blog on the line! Notice that I've used the `prod.yml` docker-compose file here, as this is production.

I've added another argument to the docker-compose command here, `-d`. This instructs docker-compose to run the containers in "detached mode", meaning you don't need to keep your console session active for the containers to continue to run.

### Point your domain at your droplet

An IP address is all fine and dandy, but it's not the easiest thing to remember. So, put your domain or subdomain in front of it!

I wanted to have the `blog.garjon.com` hostname resolve to my site. So I've gone and set my DNS records up to reflect that. Configuring your DNS records is usually relatively straight forward, but if you get stuck, a quick Google should help set you on your way.

#### My Digital Ocean DNS Records
![DNS Records](/img/using-caddy-with-hugo_dns-records.png)

## And there you have it!

Assuming all went spectacularly, you should now have a static Hugo site hosted on the line, with your domain name pointing at it and all super secure with HTTPS.

## Show me the source!

Alright alright, I've hosted the [source for this site](https://github.com/Garjon/blog) on my Github account. If something is not working for you, you may be able to find some kind of helpful hints in there.

## Where to from here?

The next step will be to look at setting up some nice CI and CD systems to help automate the propogation of changes into the wild. So stay tuned!

## Got stuck? Leave a comment!

If something in this article wasn't quite clear (or just plain incorrect!), please leave a comment. I'll try to answer any and all questions and make any corrections to this article as best I can.