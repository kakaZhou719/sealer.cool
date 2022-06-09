# Architecture

![](https://user-images.githubusercontent.com/8912557/133879086-f13e3e37-65c3-43e2-977c-e8ebf8c8fb34.png)

Sealer has two top module: Build Engine & Apply Engine

The Build Engine Using Kubefile and build context as input, and build a ClusterImage that contains all the dependencies.
The Apply Engine Using Clusterfile to init a cluster which contains kubernetes and other applications.

## Build Engine

* Parser : parse Kubefile into image metadata
* Registry : push or pull the ClusterImage
* Store : save ClusterImage to local disks

### Builders

* Lite Builder, sealer will check all the manifest or helm chart, decode docker images in those files, and cache them into ClusterImage.
* Cloud Builder, sealer will create a Cluster using public cloud, and exec `RUN & CMD` command which defined in Kubefile, then cache all the docker image in the Cluster.
* Container Builder, Using Docker container as a node, run kubernetes cluster in container then cache all the docker images.

## Apply Engine

* Infra : manage infrastructure, like create VMs in public cloud then apply the cluster on top of it. Or using docker emulation nodes.
* Runtime : cluster installer implementation, like using kubeadm to install cluster.
* Config : application config, like mysql username passwd or other configs, you can use Config overwrite any file you want.
* Plugin : plugin help us do some extra work, like exec a shell command before install, or add a label to a node after install.
* Debug : help us check the cluster is healthy or not, find reason when things unexpected.

## Other modules

* Filesystem : Copy ClusterImage rootfs files to all nodes
* Mount : mount ClusterImage all layers together
* Checker : do some pre-check and post check
* Command : a command proxy to do some tasks which os don't have the command. Like ipvs or cert manager.
* Guest : manage user application layer, like exec CMD command defined in Kubefile.
