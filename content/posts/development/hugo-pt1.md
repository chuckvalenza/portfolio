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

This post will walk you through how to set up a development server using Hugo to
serve content. It's intended for development within a LAN, and not for deployment
to a production environment. This post will only go through deployment to this
development server. Deployment to an Apache server in production will be outlined
in a following post.

[comment]: <> (to be added later once the DMZ post is created:)
[comment]: <> (It will not walk you through the network setup or deployment to
the DMZ. For an in-depth look at that, please see my post [here].)

## Setting up the development server

Hugo is a static site generator similar to Gatsby, but uses yaml and markdown
to generate pages. Building the actual site is something you can do as well,
but in the interest of time, I prefer to use pre-built themes. For this
website, I forked [toha](https://themes.gohugo.io/toha/) and made some changes.

I run debian for my client machine. While hugo is supported on Debian, the
latest version available through the debian 10 repos is `0.55.6+really0.54.0-1`.
On debian-based systems, you can check what is available with the following
command:

```console
apt-cache policy hugo
```

{{< vs 1 >}}

The toha theme had some compilation issues with this version due to some changes
in Hugo since 0.55. At the time of writing, the Arch repos have `v0.76.5` which
doesn't have these issues. For this reason, I created a simple deployment pipeline
using Git hooks on a development server running Arch linux.

Below is a diagram of the deployment pipeline. When this is set up, you will push
commits to the 'dev' server's git repository and it will automatically run a hugo
server. From here, you can check and remediate your changes before pushing to
your public repository or publishing to production.

{{< vs 3 >}}

{{< img src="/images/posts/development/hugo/pipeline.svg" width="1000" align="center">}}

{{< vs 3 >}}

This post assumes you have already set up a server with Arch linux and it's
accessible over your network.

First, we need to install a few packages on the on the server.

```console
sudo pacman -S git hugo screen
```

{{< vs 1 >}}

Once we do that, have to make two directories. I created a user and set up these
two folders in the home dir. The `site-repo` directory is where our git repo
will lie. The other is the staging area where the Hugo dev server will run.

```console
mkdir site-repo srv
```

{{< vs 1 >}}

We need to create the repository on the dev server.

```console
cd site-repo
git init --bare
```

{{< vs 1 >}}

Hugo can both compile a static site and act as a server. For our dev server,
we're going to simply spin up Hugo. To do this, we're going to use a git hook.
We need to create `~/site-repo/hooks/post-receive` so our server gets recreated
every time we pushed to repo. The Hugo server continuously watches for changes
in the files, so this shouldn't seem necessary at first glance, but the server
won't be running initially, so we would have to start it manually. What about
when we run into an runtime error on the Hugo server? Again, ssh into the box
and manually restart. To fix this, we'll run the hugo server on every push so
we can rapidly remediate server crashes. We'll put it in screen too, so we can
check on the process while it runs with `screen -ls` or view the server with
`screen -r hugo-server`.

[comment]: <> (Addresses in this file should be the full path to the file, i.e. for a
repository called `site-repo` in the home directory of `user`, the path to the
repository would be `/home/user/site-repo`. )

```bash
#!/bin/bash

ip=""
url=$ip
port=""
srv=""
repo=""
theme_repo=""

# create all of the files in our Hugo server directory
git --work-tree=$srv --git-dir=$repo checkout -f

# clone our theme only if required, else update it
cd $srv/themes/toha

if [ $((`ls -1 $srv/themes/toha | wc -l`)) -eq 0 ]; then
	git clone $theme_repo .
else
	git pull origin master --ff-only
fi

cd $srv

# kill any existing hugo servers and start the current one
screen -X -S hugo-server quit
screen -d -m -S hugo-server hugo server -w --baseURL="http://"$url":"$port"/" --bind $ip -p $port
```

{{< vs 1 >}}

Git hooks are just simply bash scripts which are executed based on their name.
We need to make this file executable.

```
chmod +x ~/site-repo/hooks/post-receive
```

{{< vs 1 >}}

Fill out the variables at the top of the script (example in parentheses):
- `ip`: the IP of the dev server (`10.10.10.10`)
- `url`: this is optional if you have DNS set up for your dev server (`dev.chuckvalenza.com`)
- `port`: the port the Hugo server should run on (`1313`)
- `srv`: the full path to the directory of the files Hugo should track (`/home/user/srv`)
- `repo`: the full path to your site's repository (`/home/user/site-repo`)
- `theme_repo`: the url to the git repository of your Hugo theme (`https://github.com/chuckvalenza/toha.git`)

{{< vs 1 >}}

Make sure the script is executable with a simple chmod and our server work is
done. We'll be able to test our script later.

## Setting up the project on your local machine

Now let's set up our Hugo project. The issue that that we're going to run into
is that the verison of Hugo on our machine may conflict with the version on the
server. Now, while it may be perfectly valid to install hugo locally, and create
the project with the version provided by debian, I opted to initiate the project
on the server. On the dev server, execute the following commands:

```console
mkdir ~/tmp
cd ~/tmp
hugo new site site -f=yaml
```

{{< vs 1 >}}

Then simply `scp` the folder from the server to your local machine and rename it
to whatever you like. Once that is copied down, you can remove the `~/tmp` folder
from the server. Let's cd into the project directory and connect it to our repo.

```console
git init
git remote add dev ssh://<user>@<server-ip>/home/<user>/site-repo
```

{{< vs 1 >}}

Now we need to add the theme as a submodule. I used
[my fork](https://github.com/chuckvalenza/toha.git). From the project directory:

```console
git submodule add https://github.com/chuckvalenza/toha.git themes/toha
```

{{< vs 1 >}}

From here, we should be able to make some modifications to the project files
and make a commit. Follow the instructions on the theme you installed for what
variables need to be added to the config.yaml and what other files need to be
modified. For the purpose of this tutorial, we're going to use the example site
provided with the theme. From your project directory, execute the following
command:

```console
cp -r themes/toha/exampleSite/* .
```

{{< vs 1 >}}

Add and commit the project files, then push to the server. If you navigate to
the server's IP address on the port you specified in the post-receive hook, you
should see our example site running.

Some limitations in this system become apparent when you are mid-commit and not
ready to publish your work or you have bugs. What I simply do is amend the
previous commit and force the new push. We don't have to worry about history
since nobody should be using this repository as their origin. Only one person
can effectively work on this at a time due to the fact that each push will
overwrite the other's work.

```console
git commit --amend && git push dev master --force
```

{{< vs 1 >}}

You should now see a site similar to the one below.

{{< vs 3 >}}

{{< img src="/images/posts/development/hugo/site.png" width="900" align="center">}}

{{< vs 3 >}}

And that is it for deployment to the development server. In part 2 of this
post, I'll describe how to deploy this to your production server.

