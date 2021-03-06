---
title: Never Worry about Updates Again with OSTree and Zincati
date: 2021-06-04
tags: []
description: Toy example to demonstrate (some of) the power of Zincati + OSTree!
tag:
  - OSTree
  - RPM-OSTree
  - Zincati
  - auto-updates
---

For a demo/toy example of how you can use Zincati with an OSTree (RPM-OSTree) -based OS, skip to the [demo section](#putting-the-two-together).

# Who cares about updates?

Great, there goes that little red badge on the Settings icon again:

> "This update provides important security and stability updates, and is recommended for all users".

Let's be honest, if it weren't for new emojis, my devices probably wouldn't be up to date. Despite how boring updates could be sometimes, oftentimes they contain important patches that are critical to the health and security of your device. This is especially true for servers. But I do wish I could just forget about updates, yet still have my machines always up-to-date...

# Recipe for auto-updates

Last time I checked, [Fedora CoreOS](https://getfedora.org/en/coreos) is one of the few Linux distributions out there that claim to support fully automatic OS updates. Having updates be automatic in itself isn't rocket science, but I think doing it reliably and frictionlessly require slightly more thought and care. 
There are a couple key points that allow Fedora CoreOS to provide automatic updates confidently:

- image-based ("immutable") nature
- transactional updates
- easy rollbacks

And on top of this, you would probably want:

- "update strategies"
- update graphs (and extensive testing)
- a long-running agent

# OSTree and RPM-OSTree

Among other things, [OSTree](https://ostreedev.github.io/ostree/) (technically now named libostree) provides us with the first three points. From the docs, "OSTree is an upgrade system for Linux-based operating systems that performs atomic upgrades of complete filesystem trees", or, colloquially, "Git for the OS".

First, an image-based system is crucial for automatic updates. This property is sometimes referred to as "immutability". An image-based system allows us to clearly and cleanly differentiate between the initial _base state_ of the operating system and user-installed packages. With a single OSTree commit, we can know exactly, bit for bit, what is in the base operating system. Colin Walters, the originator of OSTree, goes into a bit more depth in his blog post [here](https://blog.verbum.org/2020/08/22/immutable-%e2%86%92-reprovisionable-anti-hysteresis).
Being able to fully control and knowing what's in the base operating system (which should contain the most crucial packages and configuration for a distribution) between versions/releases is vital. A vendor distributing an automatically updating OS would probably want to be 100% certain of what is being delivered in an update to ensure nothing unexpected happens after the update.
And with [RPM-OSTree](https://coreos.github.io/rpm-ostree/), we have a hybrid image and package system, where we retain most of the benefits of an image-based system, but also benefit from the flexibility of traditional package-based systems.

Another key property that would make automatic updates much nicer is transactional (or atomic) updates. With automatic updates, we want the actual update process to be _"instant"_ and _reliable_, and we want our system to always be fully recoverable and in a known state even if say a system crash or power outage occurs during the process. If our server is running some workload, we don't want the packages that that workload is dependent on to be [gradually swapped out from underneath it](https://unix.stackexchange.com/questions/138214/how-is-it-possible-to-do-a-live-update-while-a-program-is-running). With transactional upgrades, we ensure that the current state of the machine is _always_ either one or the other. Although a reboot is required, the amount of time that your machine is out of service is actually minimal (the out of service time is just the time it takes for your machine to reboot). This is related to OSTree's image-based nature.

Effortless rollback is also provided by OSTree. Using Git-like content addressed storage, many "deployments" can be kept on the system without risk of using too much space, as the storage size of e.g. two different versions would only be the size of one of the images plus the delta between the two images. In this respect, the OSTree repository on the system is analogous to a Git repository. By default, Fedora CoreOS keeps two of the latest deployments, but this can be easily and efficiently scaled up to a much higher number. Therefore, we always have at least one previous working deployment to roll back to, at minimal additional cost. Micah Abbott, the CoreOS Team at Red Hat's team lead, demonstrates this in his blog post [here](https://miabbott.github.io/2019/01/22/silverblue-day-1.html#exercise-2-ostree-basics).

# Zincati

With OSTree and RPM-OSTree providing the foundation for much of the sorcery around reliable automatic updates, [Zincati](https://coreos.github.io/zincati/), an auto-update agent, fills in the gaps. It acts as a higher level controller running as a daemon process and acting as a client for RPM-OSTree.

Among other things, it can be configured with various ["update strategies"](https://coreos.github.io/zincati/usage/updates-strategy/) for coordinating when updates can occur across a fleet of machines. This is important since we want to minimize disruption in the cluster as automatic updates happen continuously over time.

It also communicates with a server that provides an update graph (via the [Cincinnati Protocol](https://coreos.github.io/zincati/development/cincinnati/protocol/)). The update graph's nodes are releases (along with useful metadata regarding the release for Zincati), and edges are legal update paths. Allowing the vendor to describe an update graph is useful for cases such as e.g. dead-end releases or "barrier" releases, both of which are present in Fedora CoreOS. A dead-end release is a release that contains critical issues that require full reprovisioning. Barrier releases fix or migrate important bits of the OS; this is [an example](https://discussion.fedoraproject.org/t/cannot-upgrade-from-32-20201104-3-0-to-newer-release/28946/4) of a barrier release in action. And perhaps most importantly, legal update edges are tested rigorously in CI by the vendor to ensure that upgrading from one node to another along an edge in the update graph is safe, secure, and seamless.

Here is a view at the [Fedora CoreOS update graph](https://builds.coreos.fedoraproject.org/graph?stream=testing) (note that it is mainly used for development at the moment).

# Putting the two together

### Setting up an OSTree repo

Let's prepare an OSTree repo in archive mode at `/var/www`. This will be the OSTree repo that we will serve over HTTP later.

```bash
$ mkdir /var/www
$ sudo ostree --repo=/var/www init --mode="archive"
```

For the purposes of this demo, we'll pull our initial commit in our repo from an existing OSTree distro. To do that, we can add e.g. the OSTree repository at `ostree.fedoraproject.org` as a remote. Of course, you can also create a functioning OSTree-based distro from scratch.

```bash
$ sudo ostree --repo=/var/www remote add \
              --contenturl=mirrorlist=https://ostree.fedoraproject.org/mirrorlist \
              --no-gpg-verify \
              fedora 'https://ostree.fedoraproject.org'
```

Then, pull in the latest commit from one of the branches. We'll use Fedora CoreOS's stable branch: `fedora/x86_64/coreos/stable`. We'll save the checksum of the latest commit of this branch in an `$original_commit` variable with the `rev-parse` command.

```bash
$ sudo ostree --repo=/var/www pull fedora fedora/x86_64/coreos/stable
$ original_commit=$(ostree --repo=/var/www rev-parse fedora/x86_64/coreos/stable)
```

From this, let's start working on our own branch, we'll name it `demo-branch`. To do so, we'll create a dummy commit with the exact same contents as `$original_commit`. We'll call the commit with this version `version1`.

```bash
$ first_commit="$(sudo ostree --repo=/var/www commit --branch='demo-branch' --tree ref="$original_commit" \
                  --add-metadata-string version='version1' --keep-metadata='fedora-coreos.stream' \
                  --keep-metadata='coreos-assembler.basearch' --parent="$original_commit")"
```

Zincati is interested in a couple metadata fields, and Zincati requires that those fields must be present in the OSTree commits that Zincati interacts with. Those metadata fields are `fedora-coreos.stream` and `coreos-assembler.basearch`. You may have noticed that `fedora-coreos.stream` is required. In principle, this isn't strictly needed and Zincati could drop the requirement in the future; Zincati currently requires it since Zincati is only used in Fedora CoreOS right now. 

In our new commit above, we simply kept the same metadata as the parent commit. The `ostree commit` command outputs the checksum of the newly created commit; we'll save it in a `$first_commit` variable.

Now, let's create an [MOTD file](https://man7.org/linux/man-pages/man5/motd.5.html) to let users booting into our new update know, at login, that the new release is branched off from Fedora CoreOS's stable stream.

```bash
mkdir -p new-tree/usr/share/misc
echo "This release is branched off from Fedora CoreOS's stable stream" > new-tree/usr/share/misc/motd
```

We can now create a new commit with this new MOTD file. We specify that our new commit is based off of `$first_commit`, and we want to add a new tree at `/path/to/new-tree`, which is a directory. As before, we keep some metadata for Zincati. We'll call the commit with this version `version2`.

```bash
$ second_commit="$(sudo ostree --repo=/var/www commit --branch='demo-branch' --base="$first_commit"  --tree=dir=/path/to/new-tree \
                   --add-metadata-string version='version2' --keep-metadata='fedora-coreos.stream' \
                   --keep-metadata='coreos-assembler.basearch' --parent="$first_commit")"
```

If we do `ostree log`, our repo should look a little something like this:

```bash
$ ostree --repo=/var/www log demo-branch
commit 3432215e0f085d888c919584fbb2b5c17b8f7f4c09583c3b30b152bc5f09a036
Parent:  b274eebaa19a2b07a1d808450a13ce077ae80553ef3e97679f83b660fc173f97
ContentChecksum:  e0d3172f2089b6366be7bb6b7166d39afef3fe60c17ecbff998f2cebc4304f8d
Date:  2021-06-05 01:21:28 +0000
Version: v2
(no subject)

commit b274eebaa19a2b07a1d808450a13ce077ae80553ef3e97679f83b660fc173f97
Parent:  10bdd7c8536c43bdd114f59a86170338e1354d0bb26dc3cd7c0e09310db27251
ContentChecksum:  e0d3172f2089b6366be7bb6b7166d39afef3fe60c17ecbff998f2cebc4304f8d
Date:  2021-06-05 00:29:16 +0000
Version: v1
(no subject)

commit 10bdd7c8536c43bdd114f59a86170338e1354d0bb26dc3cd7c0e09310db27251
Parent:  b4b2199ec09b9e4200024b52062b119035a06b3ffc27b4268c5b8c3aa6fcde17
ContentChecksum:  1a0df109c04feae5cab6a340e1ee2812f4458cebdb70bf9318108ad5188c3d1f
Date:  2021-06-01 20:18:37 +0000
Version: 34.20210518.3.0
(no subject)
```

Now all we need to do is serve our newly created OSTree repo! In this demo, we'll use the official Nginx container image to start a web server.

```bash
$ podman run -it --rm -d -p 8080:80 --name web -v /var/www:/usr/share/nginx/html nginx
```

Confirm that the server is working:

```bash
$ curl http://localhost:8080/config
[core]
repo_version=1
mode=archive-z2

[remote "fedora"]
url=https://ostree.fedoraproject.org
contenturl=mirrorlist=https://ostree.fedoraproject.org/mirrorlist
gpg-verify=false
```

### Creating an Update Graph for Zincati

Now that we have a OSTree repo ready with our two versions, we're ready to tell Zincati about it.

We will prepare an update graph containing two nodes. The node with the lower age index will be populated with our `version1` commit, and the node with the higher age index will be populated with our `version2` commit. 

Notice how we specified legal edges between updates. This is very useful for creating "barrier updates" or dead-ends. As a vendor, we should be sure to test upgrades between nodes on the two ends of edges extensively.

Also notice the `age_index` metadata field. Zincati will always try to upgrade to the _highest_ age index that the current booted deployment can possibly reach (i.e. there is an edge connecting the current booted deployment with the target version).

```bash
cat <<'EOF' > graph_template
{
  "nodes": [
    {
      "version": "",
      "metadata": {
          "org.fedoraproject.coreos.releases.age_index": "0",
          "org.fedoraproject.coreos.scheme": "checksum"
      },
      "payload": ""
    },
    {
      "version": "",
      "metadata": {
          "org.fedoraproject.coreos.releases.age_index" : "1",
          "org.fedoraproject.coreos.scheme": "checksum"
      },
      "payload": ""
    }
  ],
  "edges": [
    [
      0,
      1
    ]
  ]
}
EOF
```

We can now use `jq` to fill in the empty fields with the commit checksums and version names we saved in variables from earlier. We'll place the filled in graph also under `/var/www/` so our web server can serve it; we'll need to make sure that the path is `<server-URL>/v1/graph` in order for Zincati to find it.

```bash
$ jq --arg version1 "$version1" \
     --arg first_commit "$first_commit" \
     --arg version2 "$version2" \
     --arg second_commit "$second_commit" \
     '.nodes[0].version=$version1 | .nodes[0].payload=$first_commit | .nodes[1].version=$version2 | .nodes[1].payload=$second_commit' \
     graph_template > /var/www/v1/graph
```

### Configuring nodes

##### A bit of awkward set up for the purposes of this demo

For simplicity, we'll use the machine serving all the content described above as a client node with Zincati running on it. In the real world, of course, each client machine won't be serving itself OSTree content and update graphs.

Assuming our client machine is an RPM-OSTree-based system, we can add the OSTree repo we created above as a remote. We can then rebase to the `demo-branch`.

```bash
$ sudo ostree remote add --no-gpg-verify demo http://localhost:8080 demo-branch
$ sudo rpm-ostree rebase demo:demo-branch
⠓ Receiving objects: 94% (8631/9133) 87.1 MB/s 435.6 MB 
Receiving objects: 94% (8631/9133) 87.1 MB/s 435.6 MB... done
Staging deployment... done
...
Run "systemctl reboot" to start a reboot
$ sudo systemctl reboot
```

This part is a little awkward, since we rebased to the `demo-branch`, we've also deployed and booted into the latest commit on this branch. To see Zincati in action, we'll need to deploy and boot back into `version1`.

```bash
$ sudo rpm-ostree deploy $first_commit
...
$ sudo systemctl reboot
```

##### Zincati in action

Assuming that our nodes are starting off booted into `version1` (represented by `$first_commit`), all we need to do is configure Zincati to find the update graph we want. We can do this via a drop-in.

```bash
$ echo 'cincinnati.base_url="http://localhost:8080"' > /etc/zincati/config.d/99-cincinnati-url.toml
$ sudo systemctl restart zincati.service
```

And we're set! Zincati should soon automatically detect that we can upgrade to `version2`, and it'll handle the rest (e.g. downloading and staging `version2` in the background). Depending on the configured update strategy (`immediate` by default), Zincati will automatically reboot into our new version at some point in the future.

If something breaks, say we introduced a bug in `version2`, we can easily do `rpm-ostree rollback`. Zincati is smart enough to know not to automatically update from `version1` to `version2` again in the future.

### That's it!

Last but not least, _do_ forget to update your operating system :)

# References
- https://blog.verbum.org/2020/08/22/immutable-%e2%86%92-reprovisionable-anti-hysteresis
- https://miabbott.github.io/2019/01/22/silverblue-day-1.html
- https://ostreedev.github.io/
- https://unix.stackexchange.com/questions/138214/how-is-it-possible-to-do-a-live-update-while-a-program-is-running
