# Prerequisites

In this lab you will review the machine requirements necessary to follow this tutorial.

## Virtual Machines With Multipass

On Mac chips you have a few options to use for local VMs or running kubernetes clusters locally. Given that this tutorial is all about doing things the hard way, why don't we let the VM setup be the easy part so we can keep focus on Kubernetes. For this reason I chose multipass. Strictly Ubuntu VMs, but allows us to have the provisioning and networking set up in a single CLI command, for free - even on Mac chips.

You will need homebrew installed on your Mac for this to work [brew.sh](brew.sh).

```bash
# install multipass
$ brew install multipass
# run a default VM
$ multipass launch
# show details 
$ multipass list 
Name                    State             IPv4             Image
accessible-colobus      Running           192.168.252.2    Ubuntu 26.04 LTS
# start a bash shell inside the VM
# install and start running nginx
# in there
$ multipass shell accessible-colobus
VM $ sudo apt update
VM $ sudo apt install nginx -y
VM $ sudo systemctl enable --now nginx
# go to your browser and go to the VM
# URL from the `list` command, mine is
# 192.168.252.2:80. You should see
# the nginx landing page

# feel free to remove that test VM
$ multipass delete accessible-colobus
```

This tutorial requires four (4) virtual or physical ARM64 or AMD64 machines running Ubuntu 26.04 LTS (Resolute Raccoon). The following table lists the four machines and their CPU, memory, and storage requirements.

| Name    | Description            | CPU | RAM   | Storage |
|---------|------------------------|-----|-------|---------|
| jumpbox | Administration host    | 1   | 512MB | 10GB    |
| server  | Kubernetes server      | 1   | 2GB   | 20GB    |
| node-0  | Kubernetes worker node | 1   | 2GB   | 20GB    |
| node-1  | Kubernetes worker node | 1   | 2GB   | 20GB    |

```bash
$ multipass launch -c 1 -m 512M -d 10G -n jumpbox
$ for vm in 'server' 'node-0' 'node-1'; do multipass launch -c 1 -m 2G -d 20G -n $vm; done
```

How you provision the machines is up to you, the only requirement is that each machine meet the above system requirements including the machine specs and OS version. Once you have all four machines provisioned, verify the OS requirements by viewing the `/etc/os-release` file:

```bash
cat /etc/os-release
```

You should see something similar to the following output:

```text
PRETTY_NAME="Ubuntu 26.04 LTS"
NAME="Ubuntu"
VERSION_ID="26.04"
VERSION="26.04 LTS (Resolute Raccoon)"
VERSION_CODENAME=resolute
ID=ubuntu
ID_LIKE=debian
```

Next: [setting-up-the-jumpbox](02-jumpbox.md)
