# lab4: use vAccel from a VM

In the previous lab exercises we went through the essential steps to build
([lab1](https://github.com/nubificus/vaccel-tutorials/tree/main/lab1)), tailor
([lab2](https://github.com/nubificus/vaccel-tutorials/tree/main/lab2)) and use
([lab3](https://github.com/nubificus/vaccel-tutorials/tree/main/lab3)) the
vAccel framework.

In this lab, we go through the process of calling a vAccel operation in a
lightweight Virtual Machine and running the code in the Host. To achieve this,
we need some kind of transport layer that forwards the relevant calls from the
VM to the Host where the actual execution is going to be triggered. vAccel
supports a number of ways to do this; in this lab we are going to use 
	1. vAccel's [rpc](https://github.com/nubificus/vaccel-plugin-rpc) plugin and
	2. vAccel's [virtio](https://github.com/nubificus/vaccel-plugin-virtio) plugin.

The `rpc` plugin used to send commands to a QEMU instance running on a remote server. We are using
RPC within QEMU-emulated virtual machines for VM-to-host comminication.
The `virtio` plugin requires support from the VMM. We have ported the necessary
functionality to two popular VMMs: QEMU/KVM and AWS Firecracker.

Lets go through the process of booting a Firecracker VM and use vAccel from the
guest to do the simple vector add operation we saw in
[lab4](https://github.com/nubificus/vaccel-tutorials/tree/main/lab4). We assume
we've completed lab4 and we are in the [helper
repo](https://github.com/nubificus/vaccel-tutorial-code) base directory.

# vAccel rpc plugin

## Build vAccel and vAccel-rpc plugin

```
git clone git@github.com:nubificus/vaccel
cd vaccel
meson setup -Dauto_features=enabled build
meson compile -C build
meson install -C build
```

Clone and install `vaccel-plugin-rpc`:
```
git clone git@github.com:nubificus/vaccel-plugin-rpc.git
cd vaccel-plugin-rpc
git submodule update --init
meson setup -Drpc-agent=enabled build
meson compile -C build
meson install -C build
```
## Run agent on host
```
export VACCEL_PLUGINS=/usr/local/lib/x86_64-linux-gnu/libvaccel-noop.so
export VACCEL_LOG_LEVEL=4
export ADDRESS=tcp://127.0.0.1:65500

vaccel-rpc-agent -a "${ADDRESS}"
```

## Run an application on guest
```
export VACCEL_PLUGINS=/usr/local/lib/x86_64-linux-gnu/libvaccel-rpc.so
export VACCEL_RPC_ADDRESS=tcp://127.0.0.1:65500

classify /usr/local/share/vaccel/images/example.jpg 1
```

The output on guest is:
```
Initialized session with id: 1
classification tags: This is a dummy classification tag!
```

While on host:
```
...
[2025-03-04T12:43:34Z INFO  ttrpc::sync::server] server listen started
[2025-03-04T12:43:34Z INFO  ttrpc::sync::server] server started
[2025-03-04T12:43:34Z INFO  vaccel_rpc_agent] vAccel RPC agent started
[2025-03-04T12:43:34Z INFO  vaccel_rpc_agent] Listening on 'tcp://127.0.0.1:65500', press Ctrl+C to exit
2025.03.04-12:45:38.74 - <debug> New rundir for session 1: /run/user/1009/vaccel/vgTGoD/session.1
2025.03.04-12:45:38.74 - <debug> Initialized session 1
[2025-03-04T12:45:38Z INFO  vaccel_rpc_agent::session] Created session 1
[2025-03-04T12:45:38Z INFO  vaccel_rpc_agent::ops::genop] Genop session 1
2025.03.04-12:45:38.74 - <debug> session:1 Looking for plugin implementing VACCEL_OP_IMAGE_CLASSIFY
2025.03.04-12:45:38.74 - <debug> Returning func from hint plugin noop
2025.03.04-12:45:38.74 - <debug> Found implementation in noop plugin
2025.03.04-12:45:38.74 - <debug> [noop] Calling Image classification for session 1
2025.03.04-12:45:38.74 - <debug> [noop] Dumping arguments for Image classification:
2025.03.04-12:45:38.74 - <debug> [noop] len_img: 79281
2025.03.04-12:45:38.74 - <debug> [noop] len_out_text: 512
2025.03.04-12:45:38.74 - <debug> [noop] len_out_imgname: 512
2025.03.04-12:45:38.74 - <debug> [noop] will return a dummy result
2025.03.04-12:45:38.74 - <debug> [noop] will return a dummy result
2025.03.04-12:45:38.82 - <debug> Released session 1
[2025-03-04T12:45:38Z INFO  vaccel_rpc_agent::session] Destroyed session 1
```

If you did not install vAccel in the default directory and instead used the `--prefix=<path>` option with the Meson setup command, you must adjust the `VACCEL_PLUGINS` variable accordingly. Additionally, ensure that you export the `LD_LIBRARY_PATH` variable to point to the location of the `libvaccel.so` library.

To specify the vAccel path when installing `vaccel-plugin-rpc` repo, export `PKG_CONFIG_PATH` to the location of the vAccel pkgconfig file.
```
PKG_CONFIG_PATH=<path>/lib/x86_64-linux-gnu/pkgconfig/:$PKG_CONFIG_PATH
```

# vAccel virtio plugin
## Booting a VM

To boot a VM we need to get the VMM binary, a linux kernel and a rootfs image.
Additionally the VMM needs to support the `virtio` plugin for vAccel.
Go ahead and fetch the VMM artifacts locally:

```
# get latest release
mkdir vm-artifacts
cd vm-artifacts
wget https://s3.nbfc.io/nbfc-assets/github/vaccel/plugins/virtio/rev/main/x86_64/debug/vaccel-virtio-latest-vm.tar.xz
tar xvf vaccel-virtio-latest-vm.tar.xz
tar xvf virtio-accel-*-linux-image.tar.xz
```

The folder must contain a rootf.img and a compressed Linux kernel image (bzImage\*) with the virtio-accel module, VirtIO plugin and vAccel pre-installed.
We will use these images to emulate an entire system with QEMU. There is a Docker image with QEMU pre-installed. 
```
docker pull harbor.nbfc.io/nubificus/qemu-vaccel:x86_64
```

Let's try one of the vAccel examples, for instance image classification: `classify`.
This small program gets an image as an input and the number of 
iterations and returns the classification tag for this image. We run the
following:

```
sudo docker run  -it --privileged  --rm --mount type=bind,source="$(pwd)",destination=/data -e VACCEL_EXEC_DLCLOSE=1 -e VACCEL_PLUGINS=libvaccel-noop.so harbor.nbfc.io/nubificus/qemu-vaccel:x86_64 -r <path to vm-artifacts>/rootfs.img -k <path to vm-artifacts>/bzImage* --drive-cache -M pc --vcpus $(nproc) --cpu max --no-kvm -s qemu-$(date +"%Y%m%d-%H%M%S") -c "classify /usr/local/share/vaccel/images/example.jpg 1"
```

We see that the operation was successful and we got a the following expected output:

```
2025.03.10-12:47:43.53 - <debug> Initializing vAccel
2025.03.10-12:47:43.53 - <info> vAccel 0.6.1-186-6ef2cd74
2025.03.10-12:47:43.53 - <debug> Config:
2025.03.10-12:47:43.53 - <debug>   plugins = libvaccel-noop.so
2025.03.10-12:47:43.53 - <debug>   log_level = debug
2025.03.10-12:47:43.53 - <debug>   log_file = (null)
2025.03.10-12:47:43.53 - <debug>   profiling_enabled = false
2025.03.10-12:47:43.53 - <debug>   version_ignore = false
2025.03.10-12:47:43.53 - <debug> Created top-level rundir: /run/user/0/vaccel/sRYbUB
2025.03.10-12:47:43.53 - <info> Registered plugin noop 0.6.1-186-6ef2cd74
2025.03.10-12:47:43.53 - <debug> Registered op noop from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op blas_sgemm from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op image_classify from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op image_detect from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op image_segment from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op image_pose from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op image_depth from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op exec from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op tf_session_load from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op tf_session_run from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op tf_session_delete from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op minmax from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op fpga_arraycopy from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op fpga_vectoradd from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op fpga_parallel from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op fpga_mmult from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op exec_with_resource from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op torch_jitload_forward from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op torch_sgemm from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op opencv from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op tflite_session_load from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op tflite_session_run from plugin noop
2025.03.10-12:47:43.53 - <debug> Registered op tflite_session_delete from plugin noop
2025.03.10-12:47:43.53 - <debug> Loaded plugin noop from libvaccel-noop.so
2025.03.10-12:48:03.47 - <debug> New rundir for session 1: /run/user/0/vaccel/sRYbUB/session.1
2025.03.10-12:48:03.47 - <debug> Initialized session 1
2025.03.10-12:48:03.48 - <debug> session:1 Looking for plugin implementing VACCEL_OP_IMAGE_CLASSIFY
2025.03.10-12:48:03.48 - <debug> Returning func from hint plugin noop
2025.03.10-12:48:03.48 - <debug> Found implementation in noop plugin
2025.03.10-12:48:03.48 - <debug> [noop] Calling Image classification for session 1
2025.03.10-12:48:03.48 - <debug> [noop] Dumping arguments for Image classification:
2025.03.10-12:48:03.48 - <debug> [noop] model: (null)
2025.03.10-12:48:03.48 - <debug> [noop] len_img: 79281
2025.03.10-12:48:03.48 - <debug> [noop] len_out_text: 512
2025.03.10-12:48:03.48 - <debug> [noop] len_out_imgname: 512
2025.03.10-12:48:03.48 - <debug> [noop] will return a dummy result
2025.03.10-12:48:03.48 - <debug> [noop] will return a dummy result
2025.03.10-12:48:03.48 - <debug> Released session 1
Initialized session with id: 1
classification tags: This is a dummy classification tag!
2025.03.10-12:48:05.40 - <debug> Cleaning up vAccel
2025.03.10-12:48:05.40 - <debug> Cleaning up sessions
2025.03.10-12:48:05.40 - <debug> Cleaning up resources
2025.03.10-12:48:05.40 - <debug> Cleaning up plugins
2025.03.10-12:48:05.40 - <debug> Unregistered plugin noop
```

