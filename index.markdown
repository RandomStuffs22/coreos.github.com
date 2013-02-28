---
title: Falcore OS
layout: layout
---

# Falcore OS

Falcore OS uses the tooling built for Google Chromium OS. Therefore, if
in doubt check out the [Chromium OS Developer Guide][devguide].

[devguide]: http://www.chromium.org/chromium-os/developer-guide

# Developer Guide

## Getting Started

Lets get setup with an SDK chroot and build a bootable image of Falcore
OS. The SDK chroot has a full toolchain and isolates the build process
from quirks and differences between host OSes.

## Install depot_tools

`repo`, one of the `depot_tools`, helps to manage the collection of git
repositories that makes up Falcore OS. For install instructions visit here:
[depot tools install][depotinstall].

[depotinstall]: http://dev.chromium.org/developers/how-tos/install-depot-tools

### Bootstrap the SDK chroot

Create a project directory. This will hold all of your git repos and the SDK
chroot. A few gigs of space will be necessary.

```
mkdir coreos; cd coreos
```

Initialize the .repo directory with the manifest that describes all of the git
repos required to get started.

```
repo init -u git@github.com:falcoreos/manifest.git -g minilayout --repo-url  https://git.chromium.org/git/external/repo.git
```

Syncronize all of the required git repos from the manifest.

```
repo sync
```

### Building an image

Grab a prebuilt SDK chroot and extract it to the chroot directory.

```
wget http://storage.falcoreos.com/cros-sdk-2013.02.27.000000.tar.xz
mkdir chroot
sudo tar xf cros-sdk-2013.02.27.000000.tar.xz -C chroot
```

Enter the SDK chroot which contains all of the compilers and tooling.

```
./chromite/bin/cros_sdk
```

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

## Making Changes

### git and repo

Falcore OS is managed by `repo`. It was built for the Android project and makes
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

### Example change

## Tips and Tricks

### Searching all repo code

Using `repo forall` you can search across all of the git repos at once:

```
repo forall -c  git grep 'CONFIG_EXTRA_FIRMWARE_DIR'
```

## Base system dependency graph

Get a view into what the base system will contain and why it will contain those
things with the emerge tree view:

```
emerge-amd64-generic  --emptytree  -p -v --tree  coreos-base/coreos-dev
```

## Known Issues

### build\_packages fails on

Sometimes coreos-dev or coreos builds will fail in `build_packages` with a
backtrace pointing to `epoll`. This hasn't been tracked down but running
`build_packages` again should fix it. The error looks something like this:

```
Packages failed:
        coreos-base/coreos-dev-0.1.0-r63
        coreos-base/coreos-0.0.1-r187
```
