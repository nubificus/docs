---

title: "How to run a vAccel application using Unikraft"

date: 2021-02-03T17:15:12+02:00

author: Charalampos Mainas

description : "Instructions to use vAccel in Unikraft"

---

In this post we will describe the process of running an image classification app in Unikraft using vAccel. In order to do that we gonna need;

- a NVIDIA GPU which supports CUDA 
- the appropriate drivers
- and [jetson-inference](https://github.com/dusty-nv/jetson-inference)

As long as we have the above depedencies, we can move on setting up the vAccel environment for our example. We will need to set up 3 components:

- [vAccelrt](https://github.com/cloudkernels/vaccelrt) on the bost
- A hypervisor with vAccel support, in our case [QEMU](https://github.com/cloudkernels/qemu-vaccel/tree/vaccelrt_legacy_virtio)
- and of course [Unikraft](https://github.com/cloudkernels/unikraft/tree/vaccel)

To make things easier we have created some scripts and Docketfiles which produce a vAccel enabled QEMU and a Unikraft unikernel. You can get them from [this repo](https://github.com/nubificus/qemu-x86-build/tree/unikernels_vaccelrt). (_Note the branch vaccert\_unikernels_)

The last component can be created using the build\_guest.sh script with the -u option, like:
```
bash build_guest.sh -u
```

The first 2 components can be built using the build.sh script with the -u option, like:

```
bash build.sh -u
```

After succefully building all components we can just fire up our unikernel using the run.sh script. This script requires 2 arguments, the name of the image and the number of iterations. Note that the image should exist under `guest/qemu-guest-x86_64/data/`inside this repo. This direcotry was created by build\_guest.sh and will be shared with the docker which will execute the qemu command. An example is shown below:
```
bash run.sh dog_0.jpg 1
```

Let's take a closer look on what is happening inside the containers and how we can do each step by ourselves.

### Setting up vAccert for the host

vAccelrt is a thin and efficient runtime system that links against the user application and is responsible for dispatching operations to the relevant hardware accelerators. In our case the application is QEMU and the hardware accelerator is a NVIDIA GPU. At first let's get the source code

```
git clone https://github.com/cloudkernels/vaccelrt.git
cd vaccelrt 
git submodule update --init
```

As we said before we have a NVIDIA GPU and we will need to build the jetson plugin. Moreover we will speccify `~/.local/` as the directory where vAccelrt will be installed.

```
mkdir build && cd build
cmake -DCMAKE_INSTALL_PREFIX=/.local -DBUILD_PLUGIN_JETSON=ON ..
make && make Install
```

Now that we have vAccelrt installed we need to point which plugin to use
```
export VACCEL_BACKENDS=/.local/lib/libvaccel-jetson.so
```

### Build QEMU with vAccel support
As always at first we need to get the source code. For the time being we have a seperated branch where QEMU exposes vAccel to the guest using legacy virtio.
```
git clone https://github.com/cloudkernels/qemu-vaccel.git -b vaccelrt_legacy_virtio
cd qemu-vaccel
git submodule update --init
```

We will configure QEMU with only one target and we need to point out the install location of vAccelrt with cflags and ldflags. Please set the install prefix in case you do not want it to be installed in the default directory.
```
mkdir build && cd build
../configure --extra-cflags="-I /.local/include" --extra-ldflags="-L/.local/lib" --target-list=x86_64-softmmu --enable-virtfs
make -j12 && make install
```

That's it. We are done with the host side.

### Build the Unikraft application

in case you are bored of building more things you can just get a Unikraft image from the [releases](https://github.com/cloudkernels/unikraft_app_classify/releases/tag/v0.1) of the [example app](https://github.com/cloudkernels/unikraft_app_classify) that we will use.

We will follow the instructions from [Unikraft's documentation](http://docs.unikraft.org/users-advanced.html#) with a minor change. We will not use the official Unikraft repo but our fork. In out fork there is a vaccel branch which adds support dor vAccel.

```
git clone https://github.com/cloudkernels/unikraft.git -b vaccel
```

We will also need the example application for the image classification. We would advise to keep the same structure as Unikraft's documentation describes to reduce any modifications might needed.

```
mkdir apps/ && cd apps
git clone https://github.com/cloudkernels/unikraft_app_classify.git
cd apps/unikraft_app_classify
```

In our example there is a vaccel\_config file which enables vAccel support for Unikraft and 9pfs support to share the image with the guest. You can use that config or you can make your own config, but make sure you select vaccelrt under library configuration inside menuconfig. Furthermore there should be a way to pass the image inside the guest (9pfs etc). The currrent version of application takes as first argument the path to the image we want to classify.

```
cp vaccel_config .config && make olddefconfig
make
```

After building is done, there will be a Unikraft image for KVM under build directory.

### Fire it up

After that long journey of building let's see what we have done. Not yet...

One more thing. We need to get the model which will be used for the image classification. We will use a script which is already provided by jetson-inference (normally under `/usr/local/share/jetson-inference/tools. Also we need to point vAccel's jetson plugin the path to that model.

```
/path/to/download-models.sh
cp /usr/local/share/jetson-inference/data/networks/* networks
export VACCEL_IMAGENET_NETWORKS=/full/path/to/newly/downloaded/networks/directory
```

Finally we can run our unikernel.

```
LD_LIBRARY_PATH=/.local/lib:/usr/local/nvidia/lib:/usr/local/nvidia/lib64 qemu-system-x86_64 -cpu host -m 512 -enable-kvm -nographic -vga none \
	-fsdev local,id=myid,path=/data/data,security_model=none -device virtio-9p-pci,fsdev=myid,mount_tag=data,disable-modern=on,disable-legacy=off \
	-object acceldev-backend-vaccelrt,id=gen0 -device virtio-accel-pci,id=accl0,runtime=gen0,disable-legacy=off,disable-modern=on \
	-kernel /data/classify_kvm-x86_64 -append "vfs.rootdev=data -- dog_0.jpg 1"
```

Let's highlight some parts of the qemu command:

- LD\_LIBRARY\_PATH environment variable is needed by QEMU to dynamically link with every library it needs and in this case it also needs the vAccelrt library.
- -fsdev local,id=myid,path=./data,security\_model=none: We need to tell QEMU where will find the data that will pass to the guest. The data directory contains the image we want to classify
- -object acceldev-backend-vaccelrt,id=gen0 -device virtio-accel-pci,id=accl0,runtime=gen0,disable-legacy=off,disable-modern=on : For the vAccel driver
- Note the command line of our application which takes two arguments. The image to classify and secondly how many iterations to do
