---
title: stacker: build OCI images without host privilege
date: 2018-01-28
tags: linux, OCI, containers, golang
---

Ahoy! Recently, I've been working on a tool called
[stacker](http://github.com/anuvu/stacker), which allows unprivileged users to
build OCI images. The images that are generated are generated without uid
shifting, so they look like any other OCI image that was generated by Docker or
some other mechanism, while not requiring root (worth noting that this is what
James Bottomley has described as his motivation for writing
[shiftfs](https://lkml.org/lkml/2017/2/20/653)).

Some base setup is required in order to make this happen, though. First, you
can follow [stacker's install guide](https://github.com/anuvu/stacker#install)
to build and install it.

Next, as with any user namespaces setup, stacker needs a 65k uid delegation. On
my ubuntu VM with the ubuntu user, this looks like,

    $ grep ubuntu /etc/subuid
    ubuntu:165536:65536
    $ grep ubuntu /etc/subgid
    ubuntu:165536:65536

Note that these can be any 65k range of subuids, stacker will use whatever you
give the user you run it as.

Finally, stacker also needs a btrfs filesystem. Stacker was designed to build a
large number of varying images from a single base image, and uses btrfs to
avoid doing a large amount of i/o (and compression/decompression), undiffing
filesystems back to their original state. For the purposes of this blog post,
we can just use a loopback mounted btrfs filesystem. A slightly modified
excerpt from the stacker test suite:

    # btrfs setup
    sudo truncate -s 100G btrfs.loop
    sudo mkfs.btrfs btrfs.loop
    sudo mkdir -p roots
    # allow for unprivileged subvolume deletion; use a sane flushing strategy
    sudo mount -o user_subvol_rm_allowed,flushoncommit,loop .stacker/btrfs.loop roots
    # now make sure ubuntu can actually do stuff with this filesystem
    sudo chown -R ubuntu:ubuntu roots

And with that, we can actually run stacker and build an image:

    stacker build -f ./stacker.yaml

What goes in `stacker.yaml` you ask? Consider the example from stacker's readme:

	centos:
		from:
			type: tar
			url: http://example.com/centos.tar.gz
		environment:
			http_proxy: http://example.com:8080
			https_proxy: https://example.com:8080
		labels:
			foo: bar
			bar: baz
	boot:
		from:
			type: built
			tag: centos
		run: |
			yum install openssh-server
			echo meshuggah rocks
	web:
		from:
			type: built
			tag: centos
		import: ./lighttp.cfg
		run: |
			yum install lighttpd
			cp /stacker/lighttp.cfg /etc/lighttpd/lighttp.cfg
		entrypoint: lighthttpd
		volumes:
			- /data/db
		working_dir: /var/lib/www

The top level describes the name of a tag in the OCI image to be built, in this
case there will be three tags at the end: `centos`, `boot`, and `web` (notably,
this example is quite contrived :). Underneath those, there are the following
keys:

#. `from`: this describes the base image that stacker will start from. You can
   either start from some other image in the same stackerfile, a Docker image,
   or a tarball.
#. `import`: A set of files to download or copy into the container. Stacker
   will put these files at `/stacker`, which will be automatically cleaned up
   after the commands in the `run` section are run and the image is finalized.
#. `run`: This is the set of commands to run in order to build the image; they
   are run in a user namespaced container, with the set of files imported
   available in `/stacker`.
#. `environment`, `labels`, `working_dir`, `volumes`: these all correspond
   exactly to the similarly named bits in the [OCI image config
   spec](https://github.com/opencontainers/image-spec/blob/master/config.md#properties),
   and are available for users to pass things through to the runtime
   environment of the image.

That's a bit about stacker. Hopefully some more details about the internals
will appear at some point :). Happy hacking!