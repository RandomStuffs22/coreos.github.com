---
title: CoreOS Developer Guide
layout: layout
---

# CoreOS

[CoreOS][coreos] uses the tooling built for Google Chromium OS. Therefore, if
in doubt check out the [Chromium OS Developer Guide][devguide].

If you find bugs or need help send an email to our [mailing list][list].

[devguide]: http://www.chromium.org/chromium-os/developer-guide
[coreos]: http://www.coreos.com
[list]: https://groups.google.com/forum/#!forum/coreos-dev

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

### Updating repo manifests

The repo manifest for CoreOS lives in a git repository in
`.repo/manifests`. If you need to update the manifest edit `default.xml`
in this directory.

`repo` uses a branch called 'default' to track the upstream branch you
specify in `repo init`, this defaults to 'origin/master'. Keep this in
mind when making changes, the origin git repository should not have a
'default' branch.

## Development Workflows

### Updating Packages on an Image

Building a new VM image is time consuming process. On development images you
can use `gmerge` to build packages on your workstation and ship them to your
target VM.

1. On your workstation start the dev server inside the SDK chroot:

```
start_devserver --port 8080
```

NOTE: This port will need to be internet accessible.

2. Run /usr/local/bin/gmerge from your VM and ensure that the settings in
   `/etc/lsb-release` point to your workstation IP/hostname and port

```
/usr/local/bin/gmerge coreos-base/update_engine
```

### Updating an Image with Update Engine

If you want to test that an image you built can successfully upgrade a running
VM you can use the `--image` argument to the devserver. Here is an example:

```
start_devserver --image ../build/images/amd64-generic/latest/chromiumos_image.bin
```

From the target virtual machine you run:

```
update_engine_client -update -omaha_url $WORKSTATION_HOSTNAME:8080
```

If the update fails you can check the logs of the update engine by running:

```
journalctl -u update-engine -o cat
```

If you want to download another update you may need to clear the reboot
pending status:

```
update_engine_client -reset_status
```

## Production Workflows

### Pushing updates to the developer track

To push an update to the developer track on update.core-os.net use the following tool:

```
./build_image --board=${BOARD} --noenable_rootfs_verification dev
./core_update_developer_track.sh ../build/images/$BOARD/chromiumos_image.bin <apikey>
```

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

### Setting a default ${BOARD}

If you find using --board tiresome, particularly since amd64-generic is our
target right now, you can set it as the default for all those scripts:

```
echo amd64-generic > ~/trunk/src/scripts/.default_board
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

## Constants and IDs

### CoreOS App ID

This UUID is used to identify CoreOS to the update service and elsewhere.

```
e96281a6-d1af-4bde-9a0a-97b76e56dc57
```

### GPT UUID Types

CoreOS Root: 5dfbf5f4-2848-4bac-aa5e-0d9a20b745a6
CoreOS Reserved: c95dc21a-df0e-4340-8d7b-26cbfa9a03e0
