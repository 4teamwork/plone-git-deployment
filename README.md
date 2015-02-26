# Plone GIT deployment

[git-deploy](https://github.com/mislav/git-deploy) allows to do deployment
directly from git by pushing to the deployment remote.
This tutorial shows how to use this with Plone.


## Prerequisits

For being able to use `git-deploy` as described here, your deployment should
meet some prerequisits:

- You should have a git-repository with a policy and with buildout
  configurations.
- The policy is not distributed to index server (like pypi) but used directly
  from source.
- Using [ftw-buildouts](https://github.com/4teamwork/ftw-buildouts)'
  `production.cfg` as buildout configuration base is recommended.
- Using [ftw.upgrade](https://github.com/4teamwork/ftw.upgrade) (>=1.14.0) is
  required for automatically updating the Plone site.


## Add deployment scripts to repository

First, deployment scripts need to be added to the repository.
`git-deploy` has a command for doing, but it is optimized for Rails apps.
For Plone, the scripts can simply be downloaded like this:

```sh
$ cd the-package-repository
$ mkdir deploy
$ cd deploy
$ wget https://raw.githubusercontent.com/4teamwork/plone-git-deployment/master/deploy/after_push
$ chmod +x after_push
$ wget https://raw.githubusercontent.com/4teamwork/plone-git-deployment/master/deploy/update_plone
$ chmod +x update_plone
```

The directories `tmp` and `log` most exist in the root:

```sh
$ cd the-package-repository
$ ln -s var/log
$ mkdir tmp
$ cd tmp
$ echo "*" > .gitignore
$ echo "!.gitignore" >> .gitignore
```


## Setup remotes

For beeing able to push directly to the deployment checkout later,
we need to add one remote per deployment.
Since everybody who will deploy has to do this, it is recommended to
add a script setting it up, e.g. a `scripts/setup-git-remotes`:

```sh
#!/usr/bin/env bash

setup_remote () {
    name=$1
    url=$2

    echo ""
    echo "setup remote \"$name\" -> $url"
    git remote rm $name 2> /dev/null
    git remote add $name $url
    git fetch $name
}

setup_remote "production" "zope@my.server.ch:/home/zope/02-plone-my.website.ch-PRODUCTION"
setup_remote "testing" "zope@my.server.ch:/home/zope/02-plone-my.website.ch-TESTING"
```

```sh
$ chmod +x scripts/setup-git-remotes
```

```sh
$ cd the-package-repository
$ ./scripts/setup-git-remotes
```




## Setup GIT hooks on server

In order to set up the GIT hooks you need to install `git-deploy` locally.
The setup is done only once, so not everybody has to install git-deploy.
The deployment should already exist and buildout should have ran once.

```sh
$ gem install git-deploy
$ cd the-package-repository
$ git deploy setup -r "testing"
$ git push testing master
```


# Deploying

Now deployments can be done from the local checkout by everybody with access to
the server (without installing `git-deploy`):

```sh
$ cd the-package-repository
$ ./scripts/setup-git-remotes
$ git push testing master
```
