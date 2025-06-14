# lab5: use vAccel from a VM

In the previous lab exercises we went through the essential steps to build
([lab1](https://github.com/nubificus/vaccel-tutorials/tree/main/lab1)), tailor
([lab2](https://github.com/nubificus/vaccel-tutorials/tree/main/lab2), [lab3](https://github.com/nubificus/vaccel-tutorials/tree/main/lab3)) and use
([lab4](https://github.com/nubificus/vaccel-tutorials/tree/main/lab4)) the
vAccel framework.

In this lab, we go through the process of calling a vAccel operation in a
lightweight Virtual Machine and running the code in the Host. To achieve this,
we need some kind of transport layer that forwards the relevant calls from the
VM to the Host where the actual execution is going to be triggered. vAccel
supports a number of ways to do this; in this lab we are going to use 
	1. vAccel's [rpc](https://github.com/nubificus/vaccel-plugin-rpc) plugin and
	2. vAccel's [virtio](https://github.com/nubificus/vaccel-plugin-virtio) plugin.

In this lab, the `rpc` plugin operates over a TCP socket and is used to send commands to a QEMU instance running on a remote server. This setup enables VM-to-host communication using RPC within QEMU-emulated virtual machine.
The `virtio` plugin for vAccel implements VirtIO-based transport for vAccel operations using the `virtio-accel` kernel module. In this lab, we utilize a Docker image that includes pre-installed QEMU using an Ubuntu image with virtio-accel module, virtio-plugin and vAccel pre-installed. 

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

```
2025.03.23-13:06:34.56 - <debug> Initializing vAccel
2025.03.23-13:06:34.56 - <info> vAccel 0.6.1-194-19056528-dirty
2025.03.23-13:06:34.56 - <debug> Config:
2025.03.23-13:06:34.56 - <debug>   plugins = /usr/local/lib/x86_64-linux-gnu/libvaccel-noop.so
2025.03.23-13:06:34.56 - <debug>   log_level = debug
2025.03.23-13:06:34.56 - <debug>   log_file = (null)
2025.03.23-13:06:34.56 - <debug>   profiling_enabled = false
2025.03.23-13:06:34.56 - <debug>   version_ignore = false
2025.03.23-13:06:34.56 - <debug> Created top-level rundir: /run/user/1008/vaccel/RAIcDI
2025.03.23-13:06:34.56 - <info> Registered plugin noop 0.6.1-194-19056528-dirty
2025.03.23-13:06:34.56 - <debug> Registered op noop from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op blas_sgemm from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op image_classify from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op image_detect from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op image_segment from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op image_pose from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op image_depth from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op exec from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op tf_session_load from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op tf_session_run from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op tf_session_delete from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op minmax from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op fpga_arraycopy from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op fpga_vectoradd from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op fpga_parallel from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op fpga_mmult from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op exec_with_resource from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op torch_jitload_forward from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op torch_sgemm from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op opencv from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op tflite_session_load from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op tflite_session_run from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op tflite_session_delete from plugin noop
2025.03.23-13:06:34.56 - <debug> Registered op foo from plugin noop
2025.03.23-13:06:34.56 - <debug> Loaded plugin noop from /usr/local/lib/x86_64-linux-gnu/libvaccel-noop.so
[2025-03-23T13:06:34Z INFO  ttrpc::sync::server] server listen started
[2025-03-23T13:06:34Z INFO  ttrpc::sync::server] server started
[2025-03-23T13:06:34Z INFO  vaccel_rpc_agent] vAccel RPC agent started
[2025-03-23T13:06:34Z INFO  vaccel_rpc_agent] Listening on 'tcp://127.0.0.1:65500', press Ctrl+C to exit
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
2025.03.23-13:07:02.97 - <debug> New rundir for session 1: /run/user/1008/vaccel/RAIcDI/session.1
2025.03.23-13:07:02.97 - <debug> Initialized session 1
[2025-03-23T13:07:02Z INFO  vaccel_rpc_agent::session] Created session 1
[2025-03-23T13:07:02Z INFO  vaccel_rpc_agent::ops::genop] Genop session 1
2025.03.23-13:07:02.97 - <debug> session:1 Looking for plugin implementing VACCEL_OP_IMAGE_CLASSIFY
2025.03.23-13:07:02.97 - <debug> Returning func from hint plugin noop
2025.03.23-13:07:02.97 - <debug> Found implementation in noop plugin
2025.03.23-13:07:02.97 - <debug> [noop] Calling Image classification for session 1
2025.03.23-13:07:02.97 - <debug> [noop] Dumping arguments for Image classification:
2025.03.23-13:07:02.97 - <debug> [noop] model: (null)
2025.03.23-13:07:02.97 - <debug> [noop] len_img: 79281
2025.03.23-13:07:02.97 - <debug> [noop] len_out_text: 512
2025.03.23-13:07:02.97 - <debug> [noop] len_out_imgname: 512
2025.03.23-13:07:02.97 - <debug> [noop] will return a dummy result
2025.03.23-13:07:02.97 - <debug> [noop] will return a dummy result
2025.03.23-13:07:03.06 - <debug> Released session 1
[2025-03-23T13:07:03Z INFO  vaccel_rpc_agent::session] Destroyed session 1
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
for file in virtio-accel-*.tar.xz; do tar -xvf "$file"; done
```

The folder must contain a rootf.img and a compressed Linux kernel image (bzImage\*) with the virtio-accel module, VirtIO plugin and vAccel pre-installed.
We will use these images to emulate an entire system with QEMU. There is a Docker image with QEMU pre-installed. 
```
sudo docker pull harbor.nbfc.io/nubificus/qemu-vaccel:x86_64
```
Let's run the qemu docker image:

```
cd ..
sudo docker run  -it --privileged  --rm --mount type=bind,source="$(pwd)",destination=/data harbor.nbfc.io/nubificus/qemu-vaccel:x86_64 -r vm-artifacts/rootfs.img -k $(ls vm-artifacts/bzImage*) --drive-cache -M pc --vcpus $(nproc) --cpu max -s qemu-$(date +"%Y%m%d-%H%M%S")
```
This creates a VM under a docker container with pre-installed QEMU.
```
2025.03.24-13:22:16.16 - <debug> Initializing vAccel
2025.03.24-13:22:16.16 - <info> vAccel 0.6.1-194-19056528
2025.03.24-13:22:16.16 - <debug> Config:
2025.03.24-13:22:16.16 - <debug>   plugins = libvaccel-noop.so
2025.03.24-13:22:16.16 - <debug>   log_level = debug
2025.03.24-13:22:16.16 - <debug>   log_file = (null)
2025.03.24-13:22:16.16 - <debug>   profiling_enabled = false
2025.03.24-13:22:16.16 - <debug>   version_ignore = false
2025.03.24-13:22:16.16 - <debug> Created top-level rundir: /run/user/0/vaccel/x5HdKV
2025.03.24-13:22:16.16 - <info> Registered plugin noop 0.6.1-194-19056528
2025.03.24-13:22:16.16 - <debug> Registered op noop from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op blas_sgemm from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op image_classify from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op image_detect from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op image_segment from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op image_pose from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op image_depth from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op exec from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op tf_session_load from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op tf_session_run from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op tf_session_delete from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op minmax from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op fpga_arraycopy from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op fpga_vectoradd from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op fpga_parallel from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op fpga_mmult from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op exec_with_resource from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op torch_jitload_forward from plugin noo
2025.03.24-13:22:16.16 - <debug> Registered op torch_sgemm from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op opencv from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op tflite_session_load from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op tflite_session_run from plugin noop
2025.03.24-13:22:16.16 - <debug> Registered op tflite_session_delete from plugin noo
2025.03.24-13:22:16.16 - <debug> Loaded plugin noop from libvaccel-noop.so
```
The noop plugin is pre-exported. To use a different plugin, ensure you export it before running the application.

## Accessing the host

To run a vAccel application on guest, we need a guest-host communication. The previous step ("Booting a VM") shows us how to launch the guest, a QEMU-based VM inside a Docker container. For the host, we need to connect to the VM by executing the already running Docker container.
Open a terminal and copy the \<CONTAINER ID\> the following command returns:
```
docker ps
``` 
Run a bash shell inside the container by replacing \<CONTAINER ID\> with the ID the previous command return.
```
docker exec -it <CONTAINER ID> /bin/bash
```

Now we have two options to access the host:
1. _SSH-based communication_
2. _RPC-based communication_

### SSH-based Communication

The guest VM connects to the host over the loopback interface (`localhost`) using the SSH protocol, typically via a port that is forwarded by QEMU:
```
ssh localhost -p 60022
```
Now, you can go straight to [Run an application on guest](#run-an-application-on-guest).

### RPC-based Communication
The vAccel RPC agent runs on the host and listens for requests over vsock. Except vAccel RPC agent, the host need vAccel and an acceleration plugin. 
So, you can install vAccel and vAccel RPC agent from source following the instructions in the [Build vAccel and vAccel-rpc plugin](#build-vaccel-and-vaccel-rpc-plugin) section.

Assuming that the vaccel-rpc-agent is installed at the standard path `/usr/local/bin`, you can run the agent on host:
```
VACCEL_RPC_ADDRESS="vsock://2:2048" && VACCEL_BOOTSTRAP_ENABLED=0 vaccel-rpc-agent -a "${VACCEL_RPC_ADDRESS}" --vaccel-config "plugins=libvaccel-noop.so,log_level=4"
```
```
2025.04.11-14:47:00.13 - <debug> Initializing vAccel
2025.04.11-14:47:00.13 - <info> vAccel 0.6.1-194-19056528
2025.04.11-14:47:00.13 - <debug> Config:
2025.04.11-14:47:00.13 - <debug>   plugins = libvaccel-noop.so
2025.04.11-14:47:00.13 - <debug>   log_level = debug
2025.04.11-14:47:00.13 - <debug>   log_file = (null)
2025.04.11-14:47:00.13 - <debug>   profiling_enabled = false
2025.04.11-14:47:00.13 - <debug>   version_ignore = false
2025.04.11-14:47:00.13 - <debug> Created top-level rundir: /run/user/0/vaccel/4Eyvhe
2025.04.11-14:47:00.13 - <info> Registered plugin noop 0.6.1-194-19056528
2025.04.11-14:47:00.13 - <debug> Registered op noop from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op blas_sgemm from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op image_classify from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op image_detect from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op image_segment from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op image_pose from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op image_depth from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op exec from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op tf_session_load from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op tf_session_run from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op tf_session_delete from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op minmax from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op fpga_arraycopy from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op fpga_vectoradd from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op fpga_parallel from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op fpga_mmult from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op exec_with_resource from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op torch_jitload_forward from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op torch_sgemm from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op opencv from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op tflite_session_load from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op tflite_session_run from plugin noop
2025.04.11-14:47:00.13 - <debug> Registered op tflite_session_delete from plugin noop
2025.04.11-14:47:00.13 - <debug> Loaded plugin noop from libvaccel-noop.so
[2025-04-11T14:47:00Z INFO  ttrpc::sync::server] server listen started
[2025-04-11T14:47:00Z INFO  ttrpc::sync::server] server started
[2025-04-11T14:47:00Z INFO  vaccel_rpc_agent] vAccel RPC agent started
[2025-04-11T14:47:00Z INFO  vaccel_rpc_agent] Listening on 'vsock://2:2048', press Ctrl+C to exit
```

There is a final step before running the application. The guest, the QEMU-based docker image with the pre-installed vAccel, also needs the RPC plugin to send commands to the host agent. On guest, download the pre-built RPC plugin and export the library.
```
wget https://s3.nbfc.io/nbfc-assets/github/vaccel/plugins/rpc/rev/main/x86_64/debug/vaccel-rpc-latest-bin.tar.gz
tar xvf vaccel-rpc-latest-bin.tar.gz
rm vaccel-rpc-*.tar.gz
mv vaccel-rpc-* vaccel-rpc
export VACCEL_PLUGINS=vaccel-rpc/usr/lib/x86_64-linux-gnu/libvaccel-rpc.so
export VACCEL_RPC_ADDRESS="vsock://2:2048"
```

## Run an application on guest
Now back to the guest, let's try one of the vAccel examples, for instance image classification: `classify`.
This small program gets an image as an input and the number of 
iterations and returns the classification tag for this image. We run the
following:

```
classify /usr/local/share/vaccel/images/example.jpg 1
```

We see that the operation was successful and we got a the following expected output on guest:

```
Initialized session with id: 1
classification tags: This is a dummy classification tag!
```

While on ssh-based host we see the following output:
```
2025.03.24-13:26:37.24 - <debug> New rundir for session 1: /run/user/0/vaccel/x5HdKV
2025.03.24-13:26:37.24 - <debug> Initialized session 1
2025.03.24-13:26:37.24 - <debug> session:1 Looking for plugin implementing VACCEL_OP
2025.03.24-13:26:37.24 - <debug> Returning func from hint plugin noop
2025.03.24-13:26:37.24 - <debug> Found implementation in noop plugin
2025.03.24-13:26:37.24 - <debug> [noop] Calling Image classification for session 1
2025.03.24-13:26:37.24 - <debug> [noop] Dumping arguments for Image classification:
2025.03.24-13:26:37.24 - <debug> [noop] model: (null)
2025.03.24-13:26:37.24 - <debug> [noop] len_img: 79281
2025.03.24-13:26:37.24 - <debug> [noop] len_out_text: 512
2025.03.24-13:26:37.24 - <debug> [noop] len_out_imgname: 512
2025.03.24-13:26:37.24 - <debug> [noop] will return a dummy result
2025.03.24-13:26:37.24 - <debug> [noop] will return a dummy result
2025.03.24-13:26:37.24 - <debug> Released session 1

```

With the RPC-based host, the output shows only slight differences:
```
2025.04.11-14:48:38.02 - <debug> New rundir for session 1: /run/user/0/vaccel/4Eyvhe/session.1
2025.04.11-14:48:38.02 - <debug> Initialized session 1
[2025-04-11T14:48:38Z INFO  vaccel_rpc_agent::session] Created session 1
[2025-04-11T14:48:38Z INFO  vaccel_rpc_agent::ops::genop] Genop session 1
2025.04.11-14:48:38.02 - <debug> session:1 Looking for plugin implementing VACCEL_OP_IMAGE_CLASSIFY
2025.04.11-14:48:38.02 - <debug> Returning func from hint plugin noop
2025.04.11-14:48:38.02 - <debug> Found implementation in noop plugin
2025.04.11-14:48:38.02 - <debug> [noop] Calling Image classification for session 1
2025.04.11-14:48:38.02 - <debug> [noop] Dumping arguments for Image classification:
2025.04.11-14:48:38.02 - <debug> [noop] model: (null)
2025.04.11-14:48:38.02 - <debug> [noop] len_img: 79281
2025.04.11-14:48:38.02 - <debug> [noop] len_out_text: 512
2025.04.11-14:48:38.02 - <debug> [noop] len_out_imgname: 512
2025.04.11-14:48:38.02 - <debug> [noop] will return a dummy result
2025.04.11-14:48:38.02 - <debug> [noop] will return a dummy result
2025.04.11-14:48:38.02 - <debug> Released session 1
[2025-04-11T14:48:38Z INFO  vaccel_rpc_agent::session] Destroyed session 1

```
