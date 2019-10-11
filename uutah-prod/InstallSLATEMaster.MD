# Installation of SLATE Master

This page documents the installation of the SLATE cluster master on the SLATE Management Node reference hardware (https://slateci.io/docs/cluster/management-node.html). This document follows the installation of the uutah-prod cluster on 10-10-2019.

*VERSION INFO*

*Hardware Dell PowerEdge R440*

*Dell Service Tag 6Y37XQ2*

*iDRAC Firmware Version 3.36.36.36*

*BIOS Version 2.3.10*

*CentOS 7.7.1908 x86_64*

*Docker Version 19.03.3*

*Kubernetes v1.16.1*

*Server MGMT Hostname sl-ut-perfsonar-mgmt.chpc.utah.edu*

## Configure the iDRAC

Be sure all three network connections are properly connected and configured. The iDRAC ethernet should be placed on a private subnet. There should be a 1G mannagement connection, and a 10G ethernet data connection to the SciDMZ.

Connect a display and keyboard then power on the system. As the machine boots up be ready to press the F10 to enter the system's Lifecycle Controller. Once in the controller there will be an option to `Configure the server for remote access (iDRAC)`. From here, verify the network settings. Then scroll down until you see `User Configuration`. It is reccomended that the username is set to `slate`, then choose a secure password. You will need these credentials to manage the server through its iDRAC.

Once the iDRAC user has been configured, click back and then finish near the bottom of the screen. This should prompt you to save your changes, if it does not, then the user will not be configured.

## Verify and Update System Firmware

Connect to the iDRAC IP address that you configured in the last step (this IP will be displayed by the BIOS on reboot). 

Navigate to `System > Inventory` to view a full firmware inventory. You can manually verify versions by entering your machine's service tag at https://www.dell.com/support/home/us/en/04?app=drivers.

You can also update each component's firmware manually through the iDRAC by grabbing the download from the site linked above, and uploading it under the iDRAC's `Maintenence > System Update` tab. 

With the correct network configuration, it is possible to pull and queue all firmware updates at once through the system's Lifecycle Controller.

Enter the Lifecycle Controller (reboot and press F10). Then click `Firmware Update` and `Launch Firmware Update`, choose `Network Share(CIFS or NIFS or HTTP or HTTPS Server)`. Select HTTPS and enter `downloads.dell.com` as the Share Name/Address. Also enter `catalog` as the File Path or Update Package Path. Click next and you should be able to choose each firmware update from a list.

## Deploy the Operating System

There are many ways to provide boot media to your server. You may choose to use iPXE, physical USB drive, physical DVD drive, or another method. I have used the virtual media feature of the iDRAC.

Download the latest CentOS 7 and verify the checksum. For this install CentOS 7.7.1908 was used. Currently, a known issue prevents Kubernetes from runninng on CentOS 8. We expect this to change soon.

From the system's iDRAC navigate to `Configuration > Virtual Media` and click Connect Virtual Media. Browse for your verified CentOS ISO file, and click Map Device. Leave this interface open and Reboot your system.

Return to the Lifecycle Controller and go to `OS Deployment`. The management servers do not contain additional drives by default, so we will not be configuring RAID. Click `Go Directly to OS Deployment`. The default settings on the next page should be fine, just select Red Hat Enterprise Linux 7.6 x64 from the list and click next. You should be able to select your Virtual CD from the media list and click next and finish on the next screen. The Lifecycle Controller may complain that it couldn't validate your boot media, this is fine as long as you verified the checksum yourself.

The system will reboot and land you in the installer. Configure the OS install to your liking. My Anaconda kickstart file can be found below, for reference on how drives, ntp, and other details were defined for uutah-prod. I reccomend you manuall configure drive partitions.

For my install I configured NTP with our local time servers and I setup the local timezone. I chose to do a network install using the CHPC mirror. I did manual partitioning of the single drive using lvm for the root partition. Finally I chose to turn off kdump and make a minimal install. 

```
version=DEVEL
# System authorization information
auth --enableshadow --passalgo=sha512
# Use graphical install
graphical
# Run the Setup Agent on first boot
firstboot --enable
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=dhcp --device=em1 --ipv6=auto --activate
network  --bootproto=dhcp --device=em2 --onboot=off --ipv6=auto
network  --bootproto=dhcp --device=p2p1 --onboot=off --ipv6=auto
network  --bootproto=dhcp --device=p2p2 --onboot=off --ipv6=auto
network  --hostname=sl-ut-perfsonar-mgmt.chpc.utah.edu

# Use network installation
url --url="http://mirror.chpc.utah.edu/pub/centos/7.7.1908/os/x86_64"
# Root password
rootpw --iscrypted $6$yBPLaesr/cuqwhvw$mF3ixdRJFAdCbYBWnC3giYCCIFGQjIREMofJnlS6w9W1AulTgV.iyEOKyqyMohQQVXkxR8wE8oqEDFGzgL6P0.
# System services
services --enabled="chronyd"
# System timezone
timezone America/Denver --isUtc --ntpservers=time1.chpc.utah.edu,time2.chpc.utah.edu,time3.chpc.utah.edu
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Partition clearing information
clearpart --initlabel --list=sda3,sda2
# Disk partitioning information
part /boot/efi --fstype="efi" --noformat --onpart=sda1 --fsoptions="umask=0077,shortname=winnt"
part /boot --fstype="xfs" --ondisk=sda --size=2048
part pv.209 --fstype="lvmpv" --ondisk=sda --size=455557
volgroup centos_sl-ut-perfsonar-mgmt00 --pesize=4096 pv.209
logvol /  --fstype="xfs" --size=65536 --label="lv_root" --name=root --vgname=centos_sl-ut-perfsonar-mgmt00

%packages
@^minimal
@core
@system-admin-tools
chrony

%end

%addon com_redhat_kdump --disable --reserve-mb='auto'

%end

%anaconda
pwpolicy root --minlen=6 --minquality=1 --notstrict --nochanges --notempty
pwpolicy user --minlen=6 --minquality=1 --notstrict --nochanges --emptyok
pwpolicy luks --minlen=6 --minquality=1 --notstrict --nochanges --notempty
%end
```

Most of these details should be left to the local admin running the server. This is the proccess used to deploy uutah-prod.

## System Configruation

Following the documentation at https://slateci.io/docs/cluster/

Disable SELinux as this generally causes conflicts with Kubernetes.

`setenforce 0`

`sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux`

Swap must be disabled for Kubernetes to run effectively. For this install I chose to manually partition without creating a swap partition. If you automatically partitioned your drive swap will be enabled. To disable it run:

`swapoff -a`

`sed -e '/swap/s/^/#/g' -i /etc/fstab`

At the time of this installation we are running with firewalld disabled.

`systemctl disable --now firewalld`

It is *strongly* reccomended to disable root login over SSH.

`sed -i --follow-symlinks 's/#PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config`

Ensure that bridged network traffic goes through iptables.

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

## Install Docker

Once again following the documentation at https://slateci.io/docs/cluster/.

Install some pre-requisite software: yum-utils provides the yum-config-manager utility, and device-mapper-persistent-data and lvm2 are required by the devicemapper storage driver.

`yum install -y yum-utils device-mapper-persistent-data lvm2`

Add the docker stable repository to yum. 

`yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo`

Install the latest version of Docker communitiy and containerd.

`yum install docker-ce docker-ce-cli containerd.io -y`

Enable Docker on reboot through systemctl.

`systemctl enable --now docker`

## Install Kubernetes

Following the instructions at https://slateci.io/docs/cluster/

Relevant Kubernetes doc: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

Add the Kubernetes repository to yum.

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```

Install Kubernetes: kubeadm is a tool used to bootstrap kubernetes clusters, kubelet is the system daemon that allows the kubernetes api to control the cluster nodes, kubectl is the command line tool needed to interact with and control the cluster.

`yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes`

Enable the kubelet system daemon.

`systemctl enable --now kubelet`

## Initialize the Cluster Master With Kubdeadm

It is important that the destinction is made between the machine acting as the cluster master, and the machine acting as a cluster worker. It is generally assumed that the cluster master will run on the SLATE Management Node reference hardware. For the master we will use `kubeadm init`, but for the other nodes we will use the `kubeadm join utility`

**This portion details the steps for a machinie acting as the cluster master**

We want to initialize our cluster with the pod network CIDR specifically set to 192.168.0.0/16 this is the default range for Calico which will be our pod network. It is possible to set a different RFC1918 range during `kubeadm init` and configure Calico to use that range if needed.

`kubeadm init --pod-network-cidr=192.168.0.0/16`

If you want to permenantly enable kubectl access for the root account you will need to copy the kubernetes admin configuration (KUBECONFIG) to $HOME/.kube/config. 

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

To enable kubeconfig for a single session instead simply run:

`export $KUBECONFIG=/etc/kubernets/admin.conf`

Next setup Calico which will act as our pod network.

`kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml`

After approximately five minutes you should be able to see your Kubernetes master node is ready.

```
[root@sl-ut-perfsonar-mgmt ~]# kubectl get nodes
NAME                                 STATUS   ROLES    AGE   VERSION
sl-ut-perfsonar-mgmt.chpc.utah.edu   Ready    master   24m   v1.16.1
```

If you want to schedule pods on your master node then you must remove the master node taint. This is needed for a single node cluster.  As uutah-prod is a larger cluster the master taint was left alone. 

`kubectl taint nodes --all node-role.kubernetes.io/master-`

##  Deploy the MetalLB Load Balancer on Kubernetes

Again following https://slateci.io/docs/cluster/

Relevant Mettallb docs: https://metallb.universe.tf/installation/

Apply Metallb to our cluster. This command will create the relevant kubernetes componenents that will run our load balancer.

`kubectl apply -f https://raw.githubusercontent.com/google/metallb/v0.8.1/manifests/metallb.yaml`

Create the metallb configuration and adjust the IP range to relect your environment. These must be unallocated public IP addresses available to the machine.

```
cat <<EOF > metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 155.101.6.237-155.101.6.239
EOF

**Replace this address range with your own**
```

Create the configmap for metallb on your cluster.

`kubectl apply -f metallb-config.yaml`

## Obtain a SLATE Token

First sign into the SLATE web portal https://portal.slateci.io/login

Then navigate to https://portal.slateci.io/cli

You can use the script here to install a token. I have omitted these commands from this document.

## Install the SLATE Client

```
curl -LO https://jenkins.slateci.io/artifacts/client/slate-linux.tar.gz
tar xzf slate-linux.tar.gz
sudo mv slate /usr/local/bin
rm slate-linux.tar.gz
```

At this point you should be able to use the SLATE client to list some existing clusters.

`slate cluster list`

## Join the Cluster to the SLATE Federation

The cluster can be joined to SLATE from the master node.

`slate cluster create uutah-prod --group slate-group --org utah -y`

Update the cluster location.

`slate cluster update uutah-prod --location 40.76,-111.84`

## Add desired SLATE groups to your cluster

All groups were allowed on uutah-prod.

`slate cluster allow-group uutah-prod '*'`

## Joining Nodes to the Cluster

Additional nodes should be configured similarly. Install CentOS 7, make the system tweaks, and install docker/kubernetes. Once you have kubeadm installed on the worker nodes you can run `kubeadm join ...` to add the machine.

Run the following on the master node to get a full join command for that cluster:

`kubeadm create token --print-join-command`

The proccess of deploying worker nodes for uutah-prod is documented at ***UPDATE***

