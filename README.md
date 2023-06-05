# Image Builder + ostree Demo

This contains some basic instructions for building an ostree-based operating system using [Image Builder](https://www.osbuild.org/documentation/). These instructions should be adaptable to [Fedora IoT](https://fedoraproject.org/iot/) and [Red Hat Device Edge](https://www.redhat.com/en/technologies/device-edge)

## Prerequisites

- One system to be used for running Image Builder
  - preferably 2+ CPUs, 2G+ RAM, 40GB+ storage
- One system that will be installed with resulting OS

## Building the initial ostree commit

(This guide will not cover installing Image Builder; see the [osbuild install guide](https://www.osbuild.org/guides/user-guide/installation.html))

On the host running Image Builder, create a minimal blueprint that will be used to compose the first ostree commit.

```toml
name = "minimal"
description = "minimal fedora iot blueprint"
version = "1.0.0"
modules = []
groups = []
distro = "fedora-38"
```

Then push it to osbuild:

`composer-cli blueprints push minimal.toml`

And start the build:

`composer-cli compose start-ostree minimal iot-commit`

When the build has finished, retrieve the ostree commit tarball:

`composer-cli compose image 6f1a1eeb-4332-4767-8490-ade5980b6c03`

## Making an ostree repo + serving via HTTP

We are going to make an ostree repo that we can reuse, as we will add an updated commit later on.

First, make a directory:

`sudo mkdir -p /var/srv/httpd`

Then extract the initial ostree commit into the newly created directory:

`sudo tar -xf 6f1a1eeb-4332-4767-8490-ade5980b6c03.tar -C /var/srv/httpd`

And serve it up via an `nginx` container:

`sudo podman run --name ostree-nginx -d -v /var/srv/httpd:/usr/share/nginx/html:Z -p 8080:80 docker.io/nginx`

Confirm the content is accessible:

```
$ curl http://localhost:8080/repo/config
[core]
repo_version=1
mode=archive-z2
indexed-deltas=true
```

Additionally, you can confirm using the [`ostree log`](https://manpages.org/ostree-log) command:

```
$ ostree --repo=/var/srv/httpd/repo log fedora/38/x86_64/iot
commit f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91
ContentChecksum:  9054de3fe5f1210e3e52b38955bea0510915f89971e3b1ba121e15559d5f3a63
Date:  2023-06-04 20:01:08
Version: 38
(no subject)

```

If it is not accessible, you may need to open the port on the firewall:

`sudo firewall-cmd --add-port 8080/tcp --permanent`

### Create the installer

Create a blueprint to create the installer:

```toml
name = "iso"
description = "blueprint to create iot-installer"
version = "1.0.0"
distro = "fedora-38"

[[customizations.user]]
name = "core"
key = "ssh-rsa ...."
password = "..."
groups = ["wheel"]
```

Push it to osbuild:

`composer-cli blueprints push iso.toml`

We'll create the installer by referencing the HTTP repo we just served:

`composer-cli compose start-ostree --url http://localhost:8080/repo iso iot-installer`

And grab the resulting ISO:

`composer-cli compose image 80aa95eb-9d96-4725-9180-e08d3ad9524b`

Now use the ISO to install the OS onto the other device.

### Configuring the ostree remote

On the device where the OS was just installed, you'll need to configure the ostree remote.

The installer leaves a remote on the device thait is unusuable for future updates; confirm it looks like this:

```
$ ostree remote list -u
fedora  file:///run/install/repo/ostree/repo
```

We'll remote the "bad" remote and add a new remote that points to the new ostree repo:

```
$ sudo ostree remote delete fedora
$ sudo ostree remote add --no-gpg-verify fedora http://<Image Builder IP>:8080/repo
```

We can confirm that the remote is working with the `ostree` commands:

```
$ ostree remote refs fedora
fedora:fedora/38/x86_64/iot
$ ostree rev-parse fedora:fedora/38/x86_64/iot
f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91
```

## Building the update commit

On the system running Image Builder, update the blueprint to include the `strace` package (be sure to rev the blueprint version field):

```toml
name = "minimal"
description = "minimal fedora iot blueprint"
version = "1.0.1"
modules = []
groups = []
distro = "fedora-38"

[[packages]]
name = "strace"
version = "*"
```

And push the updated blueprint:

`composer-cli blueprints push minimal.toml`

Start the compose of the new commit and provide the `--url` and `--ref` arguments to the `composer-cli` command. This instructs the compose process to fetch the metadata from the ostree repo before starting the compose. The resulting, new ostree commit will have a reference of the original ostree commit as a parent.

`composer-cli compose start-ostree --ref fedora/38/x86_64/iot --url http://localhost:8080/repo minimal iot-commit`

When the compose is complete, fetch the tarball:

`composer-cli compose image ed3a80c5-3902-4089-99b5-da677d0b0b30`

Now extract the commit to a temporary directory. We don't want to extract to the same directory we used in the past because we want to build up the commit history in the ostree repo.

`tar -xf ed3a80c5-3902-4089-99b5-da677d0b0b30.tar -C /var/tmp`

If we inspect this new location, we can see there is a single ostree commit in the repo, but it references a parent commit:

```
$ ostree --repo=/var/tmp/repo log fedora/38/x86_64/iot
commit d523ef801e8b1df69ddbf73ce810521b5c44e9127a379a4e3bba5889829546fa
Parent:  f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91
ContentChecksum:  f0f6703696331b661fa22d97358db48ba5f8b62711d9db83a00a79b3ae0dfe16
Date:  2023-06-04 20:22:28 +0000
Version: 38
(no subject)
```

The parent commit is the same checksum from the original ostree commit we made.

In order to merge these two commits together, we need to use the [`ostree pull-local`](https://manpages.org/ostree-pull-local) command. This will copy any new metdata and content from an location on disk to a destination ostree repo. In this example, we are going to pull from the `/var/tmp` repo and target the ostree repo in `/var/srv/httpd`.

```
$ sudo ostree --repo=/var/srv/httpd/repo pull-local /var/tmp/repo
20 metadata, 22 content objects imported; 0 bytes content written
```

When we inspect the target ostree repo, we can see that there is now two commits in the repo in a logical order:

```
$ ostree --repo=/var/srv/httpd/repo log fedora/38/x86_64/iot
commit d523ef801e8b1df69ddbf73ce810521b5c44e9127a379a4e3bba5889829546fa
Parent:  f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91
ContentChecksum:  f0f6703696331b661fa22d97358db48ba5f8b62711d9db83a00a79b3ae0dfe16
Date:  2023-06-04 20:22:28 +0000
Version: 38
(no subject)

commit f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91
ContentChecksum:  9054de3fe5f1210e3e52b38955bea0510915f89971e3b1ba121e15559d5f3a63
Date:  2023-06-04 20:01:08 +0000
Version: 38
(no subject)
```

## Static Deltas and Summary files

In order to improve the network efficiency when serving updated ostree commits, we should use [static deltas](https://ostreedev.github.io/ostree/formats/#static-deltas) to reduce the size of the data being transferred over the wire. (This is probably not very effective in this example, but shows how it can be done in general.)  We need to provide a starting commit and a destination commit for the static delta generation process:

```
$ sudo ostree --repo=/var/srv/httpd/repo static-delta generate --from=f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91 --to=d523ef801e8b1df69ddbf73ce810521b5c44e9127a379a4e3bba5889829546fa
Generating static delta:
  From: f47842de7e6859cee07d743d3c67949420874727883fa9dbbaeb5824ad949d91
  To:   d523ef801e8b1df69ddbf73ce810521b5c44e9127a379a4e3bba5889829546fa
modified: 12
new reachable: metadata=20 content regular=21 symlink=1
rollsum for 0/12 modified
processing bsdiff: [0/12]
processing bsdiff: [1/12]
processing bsdiff: [2/12]
processing bsdiff: [3/12]
processing bsdiff: [4/12]
processing bsdiff: [5/12]
processing bsdiff: [6/12]
processing bsdiff: [7/12]
processing bsdiff: [8/12]
processing bsdiff: [9/12]
processing bsdiff: [10/12]
processing bsdiff: [11/12]
part 1 n:41 compressed:1560413 uncompressed:29743932
uncompressed=29743932 compressed=1560413 loose=1738650
rollsum=0 objects, 0 bytes
bsdiff=12 objects
```

Finally, we should generate a [summary file](https://ostreedev.github.io/ostree/repo/#the-summary-file), which will tell `rpm-ostree` clients the available refs and static deltas.

`sudo ostree --repo=/var/srv/httpd/repo summary -u`

## Updating the OS

Back on the system where we installed the OS, we can now check to see if our efforts worked.

First, check to see if there are updates available:

```
$ sudo rpm-ostree upgrade --check
2 metadata, 0 content objects fetched; 19 KiB transferred in 0 seconds; 0 bytes content written
Note: --check and --preview may be unreliable.  See https://github.com/coreos/rpm-ostree/issues/1579
AvailableUpdate:
        Version: 38 (2023-06-04T20:22:28Z)
         Commit: d523ef801e8b1df69ddbf73ce810521b5c44e9127a379a4e3bba5889829546fa
           Diff: 1 added
```

And if we proceed with the update, we can see that the client is fetching the static deltas we generated:

```
$ sudo rpm-ostree upgrade
⠒ Writing objects: 1...
1 delta parts, 2 loose fetched; 1544 KiB transferred in 1 seconds; 0 bytes content written
Writing objects: 1... done
Staging deployment... done
Added:
  strace-6.3-1.fc38.x86_64
Run "systemctl reboot" to start a reboot
```
