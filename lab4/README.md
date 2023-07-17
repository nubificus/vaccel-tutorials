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
supports a number of ways to do this; in this lab we are going to use vAccel's
`virtio` plugin.

The `virtio` plugin requires support from the VMM. We have ported the necessary
functionality to two popular VMMs: QEMU/KVM and AWS Firecracker.

Lets go through the process of booting a Firecracker VM and use vAccel from the
guest to do the simple vector add operation we saw in
[lab4](https://github.com/nubificus/vaccel-tutorials/tree/main/lab4). We assume
we've completed lab4 and we are in the [helper
repo](https://github.com/nubificus/vaccel-tutorial-code) base directory.

## Booting a VM

To boot a VM we need to get the VMM binary, a linux kernel and a rootfs image.
Additionally the VMM needs to support the `virtio` plugin of vAccel. To
facilitate the process we have the necessary files bundled in a [github
repo](https://github.com/nubificus/fc-x86-guest-build/releases/latest). Go
ahead and fetch them locally:

```
# get latest release
FC_RELEASE=`curl --silent "https://api.github.com/repos/cloudkernels/firecracker/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/'`
ASSETS_RELEASE=`curl --silent "https://api.github.com/repos/nubificus/fc-x86-guest-build/releases/latest" | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/'`
wget -c https://github.com/cloudkernels/firecracker/releases/download/$FC_RELEASE/firecracker-vaccel-$(uname -m)
mv firecracker-vaccel-$(uname -m) firecracker-vaccel
wget -c https://github.com/nubificus/fc-x86-guest-build/releases/download/$ASSETS_RELEASE/rootfs.img
wget -c https://github.com/nubificus/fc-x86-guest-build/releases/download/$ASSETS_RELEASE/vmlinux
chmod +x firecracker-vaccel

```

Also the `virtio`-config file(*config_virtio_accel.json*) can be retrieved from [vaccel-tutorial-code-repo](https://github.com/nubificus/vaccel-tutorial-code/blob/main/config_virtio_accel.json) 

The first thing to do is boot the VM with vAccel enabled. To be able to share
files with the VM, we'll use the network. The config file specifies a tap
interface which will be created when the VM starts, so let's create a tap
interface with the same name and give it an IP address:

```
sudo ip tuntap add dev tapFc mode tap
sudo ip addr add dev tapFc 172.42.0.1/24
sudo ip link set dev tapFc up
```

On a new terminal, we launch our firecracker VM with the
following command:

```
sudo \
VACCEL_BACKENDS=vaccelrt/build/plugins/noop/libvaccel-noop.so \
LD_LIBRARY_PATH=vaccelrt/build/src/  \
./firecracker-vaccel --api-sock fc.sock --config-file config_virtio_accel.json --seccomp-level 0
```

We are using `noop plugin` and have enabled full debug. The VM is on
`172.42.0.2`.  On the VM's console terminal, we're presented with a login
prompt, just enter `root` as the user with no password.

On our original terminal we can login to the VM via ssh (no password/auth
required). Let's try that:

```
$ ssh root@172.42.0.2
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 4.20.0 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Sat Apr 10 13:41:36 2021
root@vaccel-guest:~# 
```

Good, its working! lets exit and return to our tutorial repo base directory:

```
root@vaccel-guest:~# exit
logout
Connection to 172.42.0.2 closed.
 ~/vaccel-tutorial-code $
```

Let's try one of the vAccel examples, for instance image classification:
`classify`. This small program gets an image as an input and the number of
iterations and returns the classification tag for this image. We run the
following:

```
$ ssh root@172.42.0.2
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 4.20.0 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Sat Apr 10 14:14:13 2021
root@vaccel-guest:~# ./classify images/drone_0255.png 1
Initialized session with id: 1
Image size: 920627B
classification tags: This is a dummy classification tag!
```

We see that the operation was successful and we got a dummy string instead of
the proper classification tag. This is expected as we have used the `noop`
plugin, which just returns this dummy string. The output on the VM console is
the following:

```
[noop] Calling Image classification for session 1
[noop] Dumping arguments for Image classification:
[noop] len_img: 920627
[noop] will return a dummy result
```

Let's now try something else. Lets copy the wrapper-vaccel binary, along with
its library to the guest:

```
$ scp app/wrapper-args-vaccel app/libwrapper-args.so root@172.42.0.2:~
wrapper-args-vaccel                                                                                   100%   19KB  15.6MB/s   00:00    
libwrapper-args.so                                                                                    100%   20KB  17.0MB/s   00:00    
```

and try to run this example:

```
# ssh root@172.42.0.2
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 4.20.0 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Sat Apr 10 14:28:07 2021
root@vaccel-guest:~# LD_LIBRARY_PATH=. ./wrapper-args-vaccel 
Initialized session with id: 1
Operation successful!
A:  0  1  2  3  4  5  6  7  8  9 
+
B:  0  1  2  0  1  2  0  1  2  0 
=
C:  0  0  0  0  0  0  0  0  0  0 
```

We get no real result for the `vector_add` (array C is 0). On the VM console we
get the following:

```
[noop] Calling exec for session 1
[noop] Dumping arguments for exec:
[noop] library: /tmp/libvector_add.so symbol: vector_add
[noop] nr_read: 3 nr_write: 1
```

This means that we asked the vAccel plugin to load `/tmp/libvector_add.so`,
find the symbol `vector_add` and run it. Since we have used the `noop` plugin,
we only get debug messages. No real execution took place, so array C remains
intact. 

Let's try now the same example with the `exec` plugin. Shutdown the VM, and
re-run it, while setting `VACCEL_BACKENDS` to point to the `exec` plugin. On
the VM console terminal we do:

```
reboot # yes, FC shuts down gracefull with a reboot ;-)
rm -f fc.sock
sudo \
VACCEL_BACKENDS=vaccelrt/build/plugins/exec/libvaccel-exec.so \
LD_LIBRARY_PATH=vaccelrt/build/src/  \
./firecracker-vaccel --api-sock fc.sock --config-file config_virtio_accel.json --seccomp-level 0
```

We should be presented with a login prompt again. Login as `root` again, with
no password. Let's move to the other terminal and try to run the same example
as before:

```
$ ssh root@172.42.0.2
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 4.20.0 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

This system has been minimized by removing packages and content that are
not required on a system that users do not log into.

To restore this content, you can run the 'unminimize' command.
Last login: Sat Apr 10 14:30:51 2021
root@vaccel-guest:~# LD_LIBRARY_PATH=. ./wrapper-args-vaccel 
Initialized session with id: 1
Operation successful!
A:  0  1  2  3  4  5  6  7  8  9 
+
B:  0  1  2  0  1  2  0  1  2  0 
=
C:  0  2  4  3  5  7  6  8 10  9 
```

Aha! now we can see the result is correct, array C has been populated with the
actual `vector_add` operation result. And we can see the output in the VM
console too:

```
 Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
```

This is the `stdout`/`stderr` of our host library. If we refrain from writing
to `stdout`/`stderr`, we should see no messages in the VM console.
