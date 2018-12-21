# Plone GIT deployment

| WARNING: This project is no longer maintained. See [ftw.deploy](https://github.com/4teamwork/ftw.deploy)! |
| --- |



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

### errbit deployment notification

The ``notify_errbit`` script notifies errbit about a deployment.

For this to work, there must be a ``bin/instance0`` having these environment
variables declared:
- ``ERRBIT_URL``
- ``ERRBIT_API_KEY``
- ``ERRBIT_APP_ID`` (be aware that ``errbit-python`` does not need this)
- ``ERRBIT_ENVIRONMENT``

Installation:

```sh
$ cd the-package-repository
$ mkdir deploy
$ cd deploy
$ wget https://raw.githubusercontent.com/4teamwork/plone-git-deployment/master/deploy/notify_errbit
$ chmod +x notify_errbit
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

In order to set up the GIT hooks you need to install `git-deploy` **locally**.
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

# VPN without SSH

When the deployment is in a VPN without SSH access, we cannot push to the
deployment.
In this situation the ``deploy/pull`` script can be used for simulating a push.
It pulls from the upstream (the branch must have an upstream defined) and runs
the deployment scripts.


# Custom update script

The ``deploy/after_push`` script can be configured to run another script
than ``deploy/update_plone``.

For example you could add a ``scripts/nightly-reinstall`` and then add to
your nightly buildout configuration file:

```ini
[buildout]
deployment-update-plone-script = scripts/nightly-reinstall
```

Be aware that this must be in the ``buildout.cfg`` of the deployment (which
may be a symlink), but it can not be extended since the buildout config file
is not parsed recursively for this option.


# Zero Downtime

When upgrades need to be installed, the script normally takes the site offline
in order to prevent conflicting writes to the database while the upgrades run.

When having a zero downtime environment, such as when only a publihser writes
the database (which is stopped while running upgrades), it is safe to keep the
site running for anonymous users.

In order to enable this behavior you must set the ``deployment-zero-downtime``
option in the buildout configurations which should be upgraded in zero downtime
mode.

**WARNING: The ``deployment-zero-downtime`` must be in the ``buildout.cfg`` file
of the deployment. It does not work when using ``extend`` for this option since
the option is directly read from ``buildout.cfg``.**

Example::

```ini
[buildout]
extends =
    ...

deployment-zero-downtime = true
```

**Deploy one commit with zero downtime**

When deploying a commit with upgrade steps, the site will be taken offline
unless zero downtime is configured.
But sometimes we want to deploy a commit with (fast) upgrades to a
non-zero-downtime deployment, but without downtime.
For marking a commit as "zero-downtime proof", you can push it to the branch
`zero-downtime` on the deployment remote, before doing a regular deployment.

Example:
```sh
$ git push testing master:zero-downtime
$ git push testing master
```

**Activate zero downtime by environment variable**
When using deploy/pull, we can activate the zero downtime strategy
with an environment variable:

Example:
```sh
$ ZERO_DOWNTIME=true deploy/pull
```
