# Virtual Machines

## Introduction

Let's start building our cloud with virtual machines. You could use bare-metal machines as well, the configuration would be the same. Given that most readers (myself included) will be using virtual infrastructure, from now on I'll be referring strictly to VMs.

I chose the "[Atomic](https://www.projectatomic.io/)" CentOS/Fedora image for the VM layer because:

1. I want less responsibility for maintaining the system, including ensuring regular software updates and reboots. Atomic's idempotent nature means the OS is largely real-only, and updates/rollbacks are "atomic" (haha) procedures, which can be easily rolled back if required.
2. For someone used to administrating servers individually, Atomic is a PITA. You have to employ [tricky](https://spinningmatt.wordpress.com/2014/01/08/a-recipe-for-starting-cloud-images-with-virt-install/) [tricks](http://blog.oddbit.com/2015/03/10/booting-cloud-images-with-libvirt/) to get it to install in a non-cloud environment. It's not designed for tweaking or customizing beyond what cloud-config is capable of. For my purposes, this is good, because it forces me to change my thinking - to consider every daemon as a container, and every config as code, to be checked in and version-controlled. Atomic forces this thinking on you.
3. I want the design to be as "portable" as possible. While I run it on VPSs now, I may want to migrate it to a "cloud" provider in the future, and I'll want the most portable, reproducible design.

## Ingredients

!!! summary "Ingredients"
    3 x Virtual Machines, each with:

    * [ ] CentOS/Fedora Atomic
    * [ ] At least 1GB RAM
    * [ ] At least 20GB disk space (_but it'll be tight_)
    * [ ] Connectivity to each other within the same subnet, and on a low-latency link (_i.e., no WAN links_)


## Preparation

### Install Virtual machines

1. Install / launch virtual machines.
2. The default username on CentOS atomic is "centos", and you'll have needed to supply your SSH key during the build process.

{blurb, class:tip}
    If you're not using a platform with cloud-init support (i.e., you're building a VM manually, not provisioning it through a cloud provider), you'll need to refer to [trick #1][atomic-trick1] and [#2][atomic-trick2] for a means to override the automated setup, apply a manual password to the CentOS account, and enable SSH password logins.
{/blurb}

### Change to latest docker

Run the following on each node to replace the default docker 1.12 with docker 1.13 (_which we need for swarm mode_):
```
systemctl disable docker --now
systemctl enable docker-latest --now
sed -i '/DOCKERBINARY/s/^#//g' /etc/sysconfig/docker
```

### Enable docker experimental features

In order to be able to watch the logs of any service from any manager node, we need to enable "experimental features" in docker. (It's no longer experimental in mainstream, but under the current Atomic).

To effect this, on each node, edit **/etc/docker-latest/daemon.json**, and change from:

```
{
    "log-driver": "journald",
    "signature-verification": false
}
```

To:

```
{
    "log-driver": "journald",
    "signature-verification": false,
    "experimental": true
}
```

T>    Note the extra comma required after "false" above

Add some handy bash auto-completion for docker. Without this, you'll get annoyed that you can't autocomplete ```docker stack deploy <blah> -c <blah.yml>``` commands.

```
cd /etc/bash_completion.d/
curl -O https://raw.githubusercontent.com/docker/cli/b75596e1e4d5295ac69b9934d1bd8aff691a0de8/contrib/completion/bash/docker
```

### Tweaks

Apply some useful tweaks:

Install some useful bash aliases on each host
```
cd ~
curl -O https://gitlab.funkypenguin.co.nz/funkypenguin/geeks-cookbook-recipies/raw/master/bash/gcb-aliases.sh
echo 'source ~/gcb-aliases.sh' >> ~/.bash_profile
```

### Setup automated cleanup

```
docker run -d  \
-v /var/run/docker.sock:/var/run/docker.sock:rw \
-v /var/lib/docker:/var/lib/docker:rw  \
meltwater/docker-cleanup:latest
```

### Upgrade Atomic

Finally, apply any Atomic host updates, and reboot, by running: ```atomic host upgrade && systemctl reboot```.

### Permit connectivity between VMs

By default, Atomic only permits incoming SSH. We'll want to allow all traffic between our nodes, so add something like this to /etc/sysconfig/iptables:

```
# Allow all inter-node communication
-A INPUT -s 192.168.31.0/24 -j ACCEPT
```

And restart iptables with ```systemctl restart iptables```


!!! summary "Ready to serve..."
    After completing the above, you should have:

    * [X] 3 fresh atomic instances, at the latest releases
    * [X] Docker 1.13, with experimental features enabled
