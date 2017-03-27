## runvm

`runvm` (pronounced as `run vm`) is a CLI tool for spawning and running virtual machines using docker images. runvm 
allows you to use vastly available docker images and execute the code inside a secure virtual machine using 
standard OCI compliant docker runtime. 

## Building

`runvm` currently supports the Linux platform with various architecture support. For the purpose 
of proof of concept, it only supports launching virtual machines using KVM for now.

It must be built with Go version 1.6 or higher in order for some features to function properly.

In order to enable seccomp support you will need to install `libseccomp` on your platform.
> e.g. `libseccomp-devel` for CentOS, or `libseccomp-dev` for Ubuntu

Otherwise, if you do not want to build `runvm` with seccomp support you can add `BUILDTAGS=""` when running make.

```bash
# create a 'github.com/harche' in your GOPATH/src
cd github.com/harche
git clone https://github.com/harche/runvm
cd runvm

make
sudo make install
```

`runvm` will be installed to `/usr/local/sbin/runvm` on your system.



## Using runvm

This release supports launching virtual machines using KVM.

### Prerequisites

Ubuntu
```
apt-get install genisoimage
apt-get install qeum-kvm
```

Fedora
```
yum install genisoimage
yum install qemu-kvm qemu-img
```

Download virtual machine image from,
```
$ sudo cd /var/lib/libvirt/images   
$ sudo wget https://dl.fedoraproject.org/pub/archive/fedora/linux/releases/22/Cloud/x86_64/Images/Fedora-Cloud-Base-22-20150521.x86_64.qcow2
$ sudo mv Fedora-Cloud-Base-22-20150521.x86_64.qcow2 disk.img.orig
```
Save this in `/var/lib/libvirt/images/disk.img.orig`


### Using runvm with docker

Assuming that you have already built the `runvm` from **Building** section above, you will have to let docker 
use this new runtime so that when docker is trying to provision new container `runvm` can lanuch a virtual 
machine instead with the given docker image.

Stop the docker deamon if it's already running,
```
service docker stop
```

Execute the following command to let docker daemon know about `runvm`,
```
dockerd --add-runtime runvm=/usr/local/sbin/runvm
```

Open another shell to launch some virtual machines using docker images!

```
$ docker  run  --runtime=runvm busybox hostname
f2c647640c751414d9db7a4dffdfcf410976df2c43b7b25fed22ba41f2dd0b24
$ 
```
In above example, the command `hostname` was executed inside of a virtual machine.

```
$ sudo virsh list --all 
 Id    Name                           State
----------------------------------------------------
 116   f2c647640c751414d9db7a4dffdfcf410976df2c43b7b25fed22ba41f2dd0b24 running
$ 
```
Once the given command has completed it's execution the virtual machine is cleared 
from the system.

Note that in case you need to launch regular `cgroups` based containers all you have 
to do is to let docker use the built-in runtime `runc` that it ships with,

```
$ docker run busybox hostname
800f4cc7a69eec659b74a96c9f03165d97374b13658dfc23b885cabd3208e628
$ 
```
Docker deamon restart is *NOT* required for launching containers simultaenously using
`runc` and `runvm`


### Creating an OCI Bundle

In order to use runvm you must have your container in the format of an OCI bundle.
If you have Docker installed you can use its `export` method to acquire a root filesystem from an existing Docker container.

```bash
# create the top most bundle directory
mkdir /mycontainer
cd /mycontainer

# create the rootfs directory
mkdir rootfs

# export busybox via Docker into the rootfs directory
docker export $(docker create busybox) | tar -C rootfs -xvf -
```

After a root filesystem is populated you just generate a spec in the format of a `config.json` file inside your bundle.
`runvm` provides a `spec` command to generate a base template spec that you are then able to edit.
To find features and documentation for fields in the spec please refer to the [specs](https://github.com/opencontainers/runtime-spec) repository.

```bash
runvm spec
```

### Running Virtual Machines

Assuming you have an OCI bundle from the previous step you can execute the container in two different ways.

The first way is to use the convenience command `run` that will handle creating, starting, and deleting the container after it exits.

```bash
cd /mycontainer

runvm run mycontainerid
```

If you used the unmodified `runvm spec` template this should give you a `sh` session inside the container.


```bash
cd /mycontainer

runvm create mycontainerid

# view the container is created and in the "created" state
runvm list

# start the process inside the container
runvm start mycontainerid

# after 5 seconds view that the container has exited and is now in the stopped state
runvm list

# now delete the container
runvm delete mycontainerid
```

This adds more complexity but allows higher level systems to manage runvm and provides points in the containers creation to setup various settings after the container has created and/or before it is deleted.
This is commonly used to setup the container's network stack after `create` but before `start` where the user's defined process will be running.

### Running the test suite

`runvm` currently supports running its test suite via Docker.
To run the suite just type `make test`.

```bash
make test
```

There are additional make targets for running the tests outside of a container but this is not recommended as the tests are written with the expectation that they can write and remove anywhere.

You can run a specific test case by setting the `TESTFLAGS` variable.

```bash
# make test TESTFLAGS="-run=SomeTestFunction"
```
