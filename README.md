# CRI-O Installation

## Enable iptables for Bridge Traffic 

In order to use iptables with bridge devices, the `br_netfilter` module must be loaded into the Linux kernel. This allows the Linux kernel to perform network filtering and manipulation on packets passing through bridge interfaces.

The `overlay` module provides support for overlay filesystems, which are a type of virtual filesystem used to "overlay" one filesystem on top of another. This is commonly used in container systems, where an overlay filesystem allows multiple containers to share the same underlying image, but have separate directories for writable data. <br>
In simpler words, when a new container is created, it can start with a copy of the image and make changes to it as needed, without modifying the original image.

```
sudo modprobe overlay
sudo modprobe br_netfilter
```

### Load modules at bootup

Create the `.conf` file to load the modules at bootup so that the cri-o container runtime can be properly initialized and functional when the system starts up. This is important to ensure that containers are available and ready to run as soon as the system is fully up and running, without having to manually load the required modules each time.

```
cat <<EOF | sudo tee /etc/modules-load.d/crio.conf
overlay
br_netfilter
EOF
```

### Set sysctl parameters

Just loading the `overlay` and `br_netfilter` modules during bootup may not be enough. It may also be necessary to set specific sysctl parameters in order to ensure that the container environment is properly configured.

```
# Set up required sysctl params, these persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

### Apply sysctl params without reboot(reload)

```
sudo sysctl --system
``` 

## Disable swap

By disabling `swap`, you ensure that the Linux kernel does not use swap for memory management, which helps ensure that CRI-O operates correctly. However, if `swap` is enabled, the operating system may start swapping out memory from containers to the swap partition. This can cause unpredictable performance issues for the containers, as well as for the entire cluster.

```
sudo swapoff -a
```

### Disable swap automatically at boot time

```
(crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true
```

## Enable cri-o repositories for version 1.23

### Set variables

To install on the following operating systems, set the environment variable **$OS`** as the appropriate field in the following table:

| Operating system | $OS               |
| ---------------- | ----------------- |
| Debian Unstable  | `Debian_Unstable` |
| Debian Testing   | `Debian_Testing`  |
| Debian 11        | `Debian_11`       |
| Debian 10        | `Debian_10`       |
| Raspberry Pi OS  | `Raspbian_10`     |
| Ubuntu 22.04     | `xUbuntu_22.04`   |
| Ubuntu 21.10     | `xUbuntu_21.10`   |
| Ubuntu 21.04     | `xUbuntu_21.04`   |
| Ubuntu 20.10     | `xUbuntu_20.10`   |
| Ubuntu 20.04     | `xUbuntu_20.04`   |
| Ubuntu 18.04     | `xUbuntu_18.04`   |

```
# change supported version of cri-o for your OS 
OS="xUbuntu_20.04"
VERSION="1.23"
```

### Create APT Repository file

This file will lists package sources for the Advanced Packaging Tool (APT) used in Debian-based Linux distributions. By adding this repository file, you are specifying that packages should be installed from the specified repository for your specific OS.

```
cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /
EOF

cat <<EOF | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /
EOF
```

### Add gpg keys

 When you add a GPG key to your system, you are essentially telling your package manager to trust the packages that are signed with that key.

```
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -

curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | sudo apt-key --keyring /etc/apt/trusted.gpg.d/libcontainers.gpg add -
```

## Install cri-o and cri-o tools

```
sudo apt-get update
sudo apt-get install cri-o cri-o-runc cri-tools -y
```

### Reload systemd configurations

```
sudo systemctl daemon-reload

# enable cri-o
sudo systemctl enable crio --now
```