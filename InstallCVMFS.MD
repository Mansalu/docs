# How to Install CVMFS on a SLATE Cluster

## Prerequisites

-A Running SLATE Cluster 

-Kubectl access to your SLATE Cluster

## How to get started with squid

You can run squid as a SLATE application. For CVMFS it is reccomended that you setup a dedicated squid instance through SLATE. That instance should be given the tag 'cvmfs'.

First you will need to download the config file for squid.

`slate app get-conf osg-frontier-squid -o conf`

Now edit that confg file. You need to change the instance tag to cvmfs `Instance: cvmfs` and the service type to ClusterIP `ExternalVisibility: ClusterIP`. The correct config looks like this:

```
# Instance to label use case of Frontier Squid deployment
# Generates app name as "osg-frontier-squid-[Instance]"
# Enables unique instances of Frontier Squid in one namespace
Instance: cvmfs



Service:
  # Port that the service will utilize.
  Port: 3128
  # Controls how your service is can be accessed. Valid values are:
  # - LoadBalancer - This ensures that your service has a unique, externally
  #                  visible IP address
  # - NodePort - This will give your service the IP address of the cluster node
  #              on which it runs. If that address is public, the service will
  #              be externally accessible. Using this setting allows your
  #              service to share an IP address with other unrelated services.
  # - ClusterIP - Your service will only be accessible on the cluster's internal
  #               kubernetes network. Use this if you only want to connect to
  #               your service from other services running on the same cluster.
  ExternalVisibility: ClusterIP

SquidConf:
  # The amount of memory (in MB) that Frontier Squid may use on the machine.
  # Per Frontier Squid, do not consume more than 1/8 of system memory with Frontier Squid
  CacheMem: 128
  # The amount of disk space (in MB) that Frontier Squid may use on the machine.
  # The default is 10000 MB (10 GB), but more is advisable if the system supports it.
  # Current limit is 999999 MB, a limit inherent to helm's number conversion system.
  CacheSize: 10000
  # The range of incoming IP addresses that will be allowed to use the proxy.
  # Multiple ranges can be provided, each seperated by a space.
  # Example: 192.168.1.1/32 192.168.2.1/32
  # Use 0.0.0.0/0 for open access.
  # The default set of ranges are those defined in RFC 1918 and typically used
  # within kubernetes clusters.
  IPRange: 10.0.0.0/8 172.16.0.0/12 192.168.0.0/16
```

Once you save the configuration deploy your custom squid instance through SLATE.

`slate app install osg-frontier-squid --cluster <YOUR CLUSTER NAME> --group <YOUR GROUP NAME> --conf conf`

Now you will need to get the ClusterIP of your squid instance, you will need it later.

`kubectl get service osg-frontier-squid-cvmfs -n slate-group-<YOUR GROUP NAME>`

If you DO NOT see any services after running that command try to look at all the services in your group namespace.

`kubectl get service -n slate-group-<YOUR GROUP NAME>`

## Getting the required YAML Manifests 

`git clone https://github.com/Mansalu/prp-osg-cvmfs.git`

You'll want to use the slate branch, and you can ignore the frontier related deployment files.

`cd prp-osg-cvmfs`

`git checkout slate`

Everything you need is in `prp-osg-cvmfs/k8s/cvmfs`

## Setup the CVMFS Namespace

Everything should run in a namespace on your cluster called cvmfs.

`kubectl create namespace cvmfs`

## Setup the kubernetes configmap

We'll use a configmap to mount the default.local configuration file the provisioner needs. You will need the IP of your squid proxy instance to do this. 

Make sure you are in the `prp-osg-cvmfs/k8s/cvmfs` directory

`cd prp-osg-cvmfs/k8s/cvmfs`

Rename the template file

`mv default.local.template default.local`

Your config file must be named `default.local` or you will get an error.

Edit the configuration with your favorite text editor

`vim default.local`

Simply replace FRONTIERSQUIDSERVICE with the IP of your squid service. 

`CVMFS_HTTP_PROXY="http://FRONTIERSQUIDSERVICE:3128"`

Save and exit.

Create the kubernetes config map from the default.local configuration file we just editted.

`kubectl create configmap cvmfs-osg-config -n cvmfs --from-file=default.local`

## Deploy the Service Accounts

Again be sure you are working in the `prp-osg-cvmfs/k8s/cvmfs` directory.

`kubectl create -f accounts/`

## Deploy Provisioner and CSI proccesses 

`kubectl create -f csi-processes/`

## Setup the StorageClasses

`kubectl create -f storageclasses/`

## OPTIONAL Create Persistent Volume Claims

The HTCondor chart will automatically setup PVCs in the correct namespace. If you plan to use CVMFS outside of HTCondor you will need PVCs in the namespace where your application is running.

`kubectl create -n <YOUR DESIRED NAMESPACE> -f pvcs/`
