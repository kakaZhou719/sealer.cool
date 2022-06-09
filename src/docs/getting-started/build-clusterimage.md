# Build ClusterImage

## Build prerequisite

We use linux overlay to cache the build differ content between each layer. If your machine do not support such
feature,will build failed. you can run below cmd to check it. If not empty, just go ahead.

```shell
[root@build ~]# lsmod | grep overlay
overlay                91659  1
```

## Build command line

You can run the build command line after sealer installed. The current path is the context path ,default build type is
`lite` and use build cache.

```shell
sealer build [flags] PATH
```

Flags:

```shell
Flags:
      --base                build with base image,default value is true. (default true)
      --build-arg strings   set custom build args
  -h, --help                help for build
  -t, --imageName string    cluster image name
  -f, --kubefile string     kubefile filepath (default "Kubefile")
  -m, --mode string         cluster image build type, default is lite (default "lite")
      --no-cache            build without cache
      --platform string     set ClusterImage platform,if not set,keep same platform with runtime
```

## Build instruction

### FROM instruction

FROM: Refers to a base image, and the first instruction in the Kubefile must be a FROM instruction. If the base image is
a private repository image, the repository authentication information is required, and the sealer community also
provides an official base image for use.

> instruction format: FROM {your base image name}

Examples:

use `kubernetes:v1.19.8` which provided by sealer community as base image.

`FROM registry.cn-qingdao.aliyuncs.com/sealer-io/kubernetes:v1.19.8`

### COPY instruction

COPY: Copy files or directories in the build context to rootfs.

The cluster image file structure is based on the rootfs structure. The default target path is rootfs, and it will be
automatically created when the specified target directory does not exist.

> instruction format: COPY {src dest}

Examples:

Copy mysql.yaml to the rootfs directory.

`COPY mysql.yaml .`

Copy the executable binary "helm" to the system $PATH.

`COPY helm ./bin`

Copy remote web file or git repository to ClusterImage.

`COPY https://github.com/sealerio/applications/raw/main/cassandra/cassandra-manifest.yaml manifests`

Support wildcard copy, copy all yaml files in the test directory to rootfs manifests directory.

`COPY test/*.yaml manifests`

### ARG instruction

ARG: Supports setting command line parameters in the build phase for use with CMD and RUN instruction.

> instruction format: ARG parameter_name=default_value

Examples:

```shell
FROM kubernetes:v1.19.8
# set default version is 4.0.0, this will be used to install mongo application.
ARG Version=4.0.0
# mongo dir contains many mongo version yaml file.
COPY mongo manifests
# arg Version can be used with RUN instruction.
RUN echo ${Version}
# use Version arg to install mongo application.
CMD kubectl apply -f mongo-${Version}.yaml
```

This means run `kubectl apply -f mongo-4.0.0.yaml` in the CMD instruction.

### RUN instruction

RUN: Use the system shell to execute the build command,accept multiple command parameters, and save the command
execution result during build. If the system command does not exist, this instruction will return error.

> instruction format: RUN {command args ...}

Examples:

Use the wget command to download a kubernetes dashboard.

`RUN wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml`

### CMD instruction

CMD: Similar to the RUN instruction format, use the system shell to execute build commands. However, the CMD command
will be executed when sealer run, generally used to start and configure the cluster. In addition, unlike the CMD
instructions in the Dockerfile, there can be multiple CMD instructions in a kubefile, and support CMD command list.

> instruction format: CMD {command args ...}

Examples:

Install a kubernetes dashboard using the kubectl command.

`CMD kubectl apply -f recommended.yaml`

Install mysql,redis and another saas application use one CMD command.

`CMD kubectl apply -f mysql, kubectl apply -f redis, kubectl apply -f saas`

## Build type

Currently, sealer build only supports lite build for different requirement scenarios.

### lite build mode

The lightest build mode. By parsing the helm Chart, submitting the image list, parsing the kubernetes resource file
under the manifest to build the ClusterImage. and this can be done without starting the cluster

The advantages of this build mode is the lowest resource consumption . Any host installed sealer can use this mode to
build ClusterImage.

The disadvantage is that some scenarios cannot be covered. For example, the image deployed through the operator cannot
be obtained, and some images delivered through proprietary management tools are also can not be used.

In addition, some command such as `kubectl apply` or `helm install` will execute failed because the lite build process
will not pull up the cluster, but it will be saved as a layer of the image in the build stage.

Lite build is suitable for the scenarios where there is a list of known images or no special resource requirements.

Kubefile example:

```shell
FROM kubernetes:v1.19.8
COPY imageList manifests
COPY apollo charts
COPY helm /bin
CMD helm install charts/apollo
COPY recommended.yaml manifests
CMD kubectl apply -f manifests/recommended.yaml
```

As in the above example, the lite build will parse and cache the list of images to the registry from the following three
locations:

* `manifests/imageList`: The content is a list of images line by line, If this file exists, will be extracted to the
  desired image list . The file name of `imageList` must be fixed, unchangeable, and must be placed under manifests.

* `manifests` directory: Lite build will parse all the yaml files in the manifests directory and extract it to the
  desired image list.

* `charts` directory: this directory contains the helm chart, and lite build will resolve the image address from the
  helm chart through the helm engine.

```shell
sealer build -t my-cluster:v1.19.9 .
```

## Build arg

If the user wants to customize some parameters in the build stage, or in the image startup stage. could
set `--build-arg` or write `ARG` in the Kubefile.

### used build arg in Kubefile

examples:

```shell
FROM kubernetes:v1.19.8
# set default version is 4.0.0, this will be used to install mongo application.
ARG Version=4.0.0
# mongo dir contains many mongo version yaml file.
COPY mongo manifests
# arg Version can be used with RUN instruction.
RUN echo ${Version}
# use Version arg to install mongo application.
CMD kubectl apply -f mongo-${Version}.yaml
```

this will use `ARG` value 4.0.0 to build the image.

```shell
sealer build -t my-mongo:v1 .
```

### use build arg in sealer build command line

examples:

use `--build-arg` value to overwrite the `ARG` value set in the kuebfile. this will install mongo application with
version 4.0.7.

```shell
sealer build -t my-mongo:v1 --build-arg Version=4.0.7 .
```

### use build arg in sealer run command line

examples:

use `--cmd-args` to overwrite the `ARG` value of CMD instruction set in the kuebfile. this will install mongo
application equals run  `kubectl apply -f mongo-5.1.1.yaml`.

```shell
sealer run --cmd-args Version=5.1.1 -m 172.16.0.227 -p passsword my-mongo:v1
```

### use build arg in Clusterfile

examples:

use `cmd_args` fields to overwrite the `ARG` value of CMD instruction set in the kuebfile. this will install mongo
application equals run  `kubectl apply -f mongo-4.9.0.yaml`.

```yaml
apiVersion: sealer.cloud/v2
kind: Cluster
metadata:
  creationTimestamp: null
  name: my-cluster
spec:
  cmd_args:
    - Version=4.9.0
  hosts:
    - ips:
        - 172.16.0.227
  image: my-mongo:v1
  ssh:
    passwd: passsword
    pk: /root/.ssh/id_rsa
    port: "22"
    user: root
```

```shell
sealer apply -f Clusterfile
```

## More build examples

### lite build:

`sealer build -f Kubefile -t my-kubernetes:1.19.8`

### build without cache:

`sealer build -f Kubefile -t my-kubernetes:1.19.8 --no-cache`

### build without base:

`sealer build -f Kubefile -t my-kubernetes:1.19.8 --base=false`

### build with args:

`sealer build -f Kubefile -t my-kubernetes:1.19.8 --build-arg MY_ARG=abc,PASSWORD=Sealer123`

### build with different platform:

`sealer build -f Kubefile -t my-kubernetes:1.19.8 --platform linux/arm64,linux/amd64`

note that: if you want to copy different platform binary,refer to the examples

Kubefile:

```shell
FROM kubernetes:v1.19.8
COPY dashborad.yaml manifests
COPY ${ARCH}/helm bin # copy binary file,make sure the build context have the same number platform binary files.
COPY my-mysql charts
CMD helm install my-mysql bitnami/mysql --version 8.8.26
CMD kubectl apply -f manifests/dashborad.yaml
```

build context tree:

```yaml
├── amd64
│   └── helm
├── arm64
│   └── helm
├── dashboard.yaml
├── Kubefile
└── my-mysql
```

sealer build cmd line:

`sealer build --platform linux/arm64,linux/amd64 -t kubernetes-multi-arch:v1.19.8`

### build with private image registry

#### different registry have different users

just to login,for example :

`sealer login registry.cn-qingdao.aliyuncs.com -u username -p password`

#### same registry have different users

you need to write the credential file named at "imageListWithAuth.yaml" in your build context. and its format like
below, it is still possible to trigger sealer build to pull docker images, works like `COPY imageList manifests`.

```yaml
- registry: registry.cn-shanghai.aliyuncs.com
  username: user1
  password: pw
  images:
    - registry.cn-shanghai.aliyuncs.com/xxx/xxx1:v1.1
    - registry.cn-shanghai.aliyuncs.com/xxx/xxx2:v1.1
- registry: registry.cn-shanghai.aliyuncs.com
  username: user2
  password: pw
  images:
    - registry.cn-shanghai.aliyuncs.com/xxx/xxx3:v1.1
    - registry.cn-shanghai.aliyuncs.com/xxx/xxx4:v1.1
```

filed "registry" is optional , if not present sealer will use the default "docker.io" as its domain name. below is
example build context: this will trigger pull images form `imageList` and `imageListWithAuth.yaml`.

For example:

```shell
[root@iZbp16ikro46xwgqzij67sZ build]# ll
total 12
-rw-r--r-- 1 root root   7 Feb 28 14:10 imageList
-rw-r--r-- 1 root root 450 Mar  1 10:20 imageListWithAuth.yaml
-rw-r--r-- 1 root root  49 Feb 28 14:06 Kubefile
[root@iZbp16ikro46xwgqzij67sZ build]#
[root@iZbp16ikro46xwgqzij67sZ build]# cat Kubefile
FROM kubernetes:v1.19.8
COPY imageList manifests
```

## Base image list

### base image without network plugin

platform of base ClusterImage support both amd64 and arm64.

|image name | kubernetes version|docker version|
--- |  ---| ---|
|kubernetes:v1.19.8-alpine|1.19.8|19.03.14|

### base image with sealer docker and calico

platform of base ClusterImage support both amd64 and arm64.

|image name | kubernetes version|docker version|
--- |  ---| ---|
|kubernetes:v1.19.8| 1.19.8|19.03.14|

### base image with native docker

|image name |platform| kubernetes version|docker version|
--- | --- | ---| ---|
|kubernetes-kyverno:v1.19.8|AMD| 1.19.8|19.03.15|