---
title: "Deploying a staticly generated site with Hugo - Part 1"
date: 2020-10-19T17:06:06-07:00
hero: /images/posts/development/hugo/hugo.svg
description: Setting up a Hugo development pipeline
theme: Toha
author:
  name: Chuck Valenza
  image: /images/avatar.svg
menu:
  sidebar:
    name: Hugo - pt 1
    identifier: hugo-pt1
    parent: development
    weight: 500
---

## Server Setup

Hugo is a static site generator similar to Gatsby, but uses yaml and markdown
to generate pages. Building the actual site is something you can do as well,
but in the interest of time, I prefer to use pre-built themes. For this
website, I forked [toha](https://themes.gohugo.io/toha/) and made some changes.

I run debian for my client machine. While hugo is supported on Debian, the
latest version available through the debian 10 repos is `0.55.6+really0.54.0-1`.
On debian-based systems, you can check what is available with the following
command:

```
apt-cache policy hugo
```

The toha theme had some compilation issues with this version due to some changes
in Hugo since 0.55. At the time of writing is Arch repos have `v0.76.5` which
doesn't have these issues. For this reason, I created a simple deployment pipeline
using Git hooks on an Arch server.

Below is a diagram of the deployment pipeline.

{{< vs 3 >}}

{{< img src="/images/posts/development/hugo/pipeline.png" width="900" align="center">}}

{{< vs 3 >}}

This post assumes you have already set up a machine with Arch linux and it's
accessible over your network. This post will only go through deployment to the
development server.

First, we need to install a few packages on the on the server.

```
$ sudo pacman -S git hugo screen
```

Once we do that, have to make two directories. I created a user and set up these
two folders in the home dir. The `site-repo` directory is where our git repo
will lie. The other is the staging area where the Hugo dev server will run.

```
mkdir site-repo srv
```

We need to create the repository on the dev server.

```
cd site-repo
git init --bare
```

Hugo can both compile a static site and act as a server. For our dev server,
we're going to simply spin up Hugo. To do this, we're going to use a git hook.
We need to create `~/site-repo/hooks/post-receive` so our server gets recreated
every time we pushed to repo. The Hugo server continuously watches for changes
in the files, so this shouldn't seem necessary at first glance, but the server
won't be running initially, so we would have to start it manually. What about
when we run into an runtime error on the Hugo server? Again, ssh into the box
and manually restart. To fix this, we'll run the hugo server on every push so
we can rapidly remediate server crashes. We'll put it in screen too, so we can
check on the process while it runs.

```
#!/bin/bash

ip=""
url=$ip
srv=""
stage=""

git --work-tree=$srv --git-dir=$stage checkout -f
cd $srv
screen -X -S hugo-server quit
screen -d -m -S hugo-server hugo server -w --baseURL="http://"$url":1313/" --bind $ip
```

Insert the IP of the dev server, the name for your staging area (`site-repo` in
this case) in `stage`, and the directory of the Hugo server files in `srv`. Make
sure the script is executable with a simple chmod and our server work is done.
