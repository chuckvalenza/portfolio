# Deployment

I run hugo on an Arch machine to ensure I'm using the latest version of Hugo.
At the time of writing this, the latest verion available on pacman is
`v0.76.2/extended`.

First, create the necessary directories and a bare git repo on the server. The
hook uses the ~/srv directory to compile the static files and stand up a hugo
test server.

```
cd && mkdir site-repo srv
cd ~/site-repo && git init --bare
```

Add the post-receive hook to the `site-repo/hooks/` directory

Install dependencies

```
pacman -S openssh screen git hugo vim
```

Add the remote

```
git remote add dev-srv ssh://user@<ip>/full/path/to/site-repo/
```

