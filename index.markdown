---
title: CoreOS Developer Guide
layout: layout
---

# CoreOS

CoreOS uses the tooling built for Google Chromium OS. Therefore, if
in doubt check out the [Chromium OS Developer Guide][devguide].

[devguide]: http://www.chromium.org/chromium-os/developer-guide

# Developer Guide

## Getting Started

Lets get setup with an SDK chroot and build a bootable image of Core
OS. The SDK chroot has a full toolchain and isolates the build process
from quirks and differences between host OSes.

## Install depot_tools

`repo`, one of the `depot_tools`, helps to manage the collection of git
repositories that makes up CoreOS. Pull down the code and add it to your
path:

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH":`pwd`/depot_tools
```

You may want to add this to your .bashrc or /etc/profile.d/ so that you don’t
need to reset your $PATH manually each time you open a new shell.

### Bootstrap the SDK chroot

Create a project directory. This will hold all of your git repos and the SDK
chroot. A few gigs of space will be necessary.

```
mkdir coreos; cd coreos
```

Initialize the .repo directory with the manifest that describes all of the git
repos required to get started.

```
repo init -u https://github.com/coreos/manifest.git -g minilayout --repo-url  https://git.chromium.org/git/external/repo.git
```

Syncronize all of the required git repos from the manifest.

```
repo sync
```

### Building an image

Download and enter the SDK chroot which contains all of the compilers and
tooling.

```
./chromite/bin/cros_sdk
```

**WARNING:** If you evern need to delete the SDK chroot use
`./chromite/bin/cros_sdk --delete`. Otherwise, you will delete `/dev`
entries that are bind mounted into the chroot.

Setup the "core" user's password.

```
./set_shared_user_password.sh
```

Target amd64-generic for this image.

```
export BOARD=amd64-generic
```

Setup a board root filesystem in /build/${BOARD}

```
./setup_board --board=${BOARD}
```

Build all of the target binary packages

```
./build_packages --board=${BOARD}
```

Build an image based on the built binary packages along with the developer
overlay.

```
./build_image --board=${BOARD} --noenable_rootfs_verification dev
```

After this finishes up commands for converting the raw bin into
a bootable vm will be printed. Run the `image_to_vm` command.

### Booting

Once you build an image you can launch it with KVM (instructions will
print out after image_to_vm.sh runs).

To demo the general direction we are starting in now the OS starts two
small daemons that you can access over an HTTP interface. The first,
systemd-rest, allows you to stop and stop units via HTTP. The other is a
small server that you can play with shutting off and on called
motd-http. You can try these daemons with:

```
curl http://127.0.0.1:8000
curl http://127.0.0.1:8080/units/motd-http.service/stop/replace
curl http://127.0.0.1:8000
curl http://127.0.0.1:8080/units/motd-http.service/start/replace
```

## Making Changes

### git and repo

CoreOS is managed by `repo`. It was built for the Android project and makes
managing a large number of git repos easier, from the announcement blog:

> The repo tool uses an XML-based manifest file describing where the upstream
> repositories are, and how to merge them into a single working checkout. repo
> will recurse across all the git subtrees and handle uploads, pulls, and other
> needed items. repo has built-in knowledge of topic branches and makes working
> with them an essential part of the workflow. 
> -- via the [Google Open Source Blog][repo-blog]

[repo-blog]: http://google-opensource.blogspot.com/2008/11/gerrit-and-repo-android-source.html

You can find the full manual for repo by visiting [Version Control with Repo and Git][vc-repo-git].

[vc-repo-git]: http://source.android.com/source/version-control.html

## Tips and Tricks

### Searching all repo code

Using `repo forall` you can search across all of the git repos at once:

```
repo forall -c  git grep 'CONFIG_EXTRA_FIRMWARE_DIR'
```

### Caching git https passwords

Note: You need git 1.7.10 or newer to use the credential helper

Turn on the credential helper and git will save your password in memory
for some time:

```
git config --global credential.helper cache
```

Why doesn't coreos use SSH in the git remotes? Because, we can't do
anonymous clones from github with a ssh URL. In the future we will fix
this.

### Base system dependency graph

Get a view into what the base system will contain and why it will contain those
things with the emerge tree view:

```
emerge-amd64-generic  --emptytree  -p -v --tree  coreos-base/coreos-dev
```

### SSH Config

You will be booting lots of VMs with on the fly ssh key generation. Add
this in your `$HOME/.ssh/config` to stop the annoying fingerprint warnings.

```
Host 127.0.0.1
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  User core
  LogLevel QUIET
```

## Known Issues

### build\_packages fails on coreos-base

Sometimes coreos-dev or coreos builds will fail in `build_packages` with a
backtrace pointing to `epoll`. This hasn't been tracked down but running
`build_packages` again should fix it. The error looks something like this:

```
Packages failed:
        coreos-base/coreos-dev-0.1.0-r63
        coreos-base/coreos-0.0.1-r187
```
