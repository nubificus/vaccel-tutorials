#
FPGA setup:
#
Note:
The following notes work with linux but is also tested on petalinux as well, if you are using a Xilinx platform.

**Wifi/Internet:**

On the PYNQ boards, there were many issues regarding connectivity and internet access with the board itself.
Solutions will be different depending on your network setup, but generally bridging connections across network connections on ethernet would be fine normally.

If for whatever reason - could be due to policy and just the way the network is setup for your particular setup, a static IP cannot be assigned for your ethernet, many issues will occur.

Bridging a network creates a new network and will serve as a DHCP IP address, and you cannot longer connect to the board's static IP (for this particular board it was 192.168.2.99). I would like to say its a PNYQ issue, but its really a hardware/setup issue and the easiest way I found was to use a USB Wifi dongle or to plug it into a router if you have access.

For USB Wifi dongle:

* Make sure you are running a linux image

* After pulling up the terminal window:
```python
$python3
Python 3.6.5 (default, Apr  1 2018, 05:46:30)
[GCC 7.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.

>>> from pynq.lib import Wifi
>>> port = Wifi()
>>> port.connect('network_name', 'network_password', True)
```

Working with TP-LINK TL-WN823N but from other sources online the following work as well:

* Wi-Pi from element14 (FCC ID: OYR-COMFAST88)
* EASTECH Ralink RT5370 Raspberry PI WiFi Adapter.

Other modules may not work due to kernel configurations.

**Lab1**:

```c
diff --git a/src/resources.c b/src/resources.c
index 1255c3b..859a8e4 100644
--- a/src/resources.c
+++ b/src/resources.c
@@ -169,7 +169,7 @@ int resource_create_rundir(struct vaccel_resource *res)
        const char *root_rundir = vaccel_rundir();
        char rundir[MAX_RESOURCE_RUNDIR];
-       int len = snprintf(rundir, MAX_RESOURCE_RUNDIR, "%s/resource.%lu",
+       int len = snprintf(rundir, MAX_RESOURCE_RUNDIR, "%s/resource.%llu",
                        root_rundir, res->id);
        if (len == MAX_RESOURCE_RUNDIR) {
                vaccel_error("rundir path '%s/resource.%lu' too long",
```
lets you compile the rest as fine, throws some errors, but haven't found any issues yet.

Also might need a -c at the end:
> gcc -I ../src/include/ -I ../third-party/slog/src ../examples/noop.c -c

**Lab2**:

This is the initial error I got on this board:

```
CMake Error at plugins/helloworld/CMakeLists.txt:8 (install):
  install TARGETS given no LIBRARY DESTINATION for shared library target
  "vaccel-helloworld".
```

First make sure pthreads and pthreads_create exist first:

In a new directory:

>../test/CMakeLists.txt

```
cmake_minimum_required (VERSION 2.8.7)
find_package(Threads)
```
And then:
```
$ mkdir build
$ cd build
$ cmake ../
```
Results in:
```
-- The C compiler identification is GNU 7.3.0
-- The CXX compiler identification is GNU 7.3.0
-- Check for working C compiler: /usr/bin/cc
-- Check for working C compiler: /usr/bin/cc -- works
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Detecting C compile features
-- Detecting C compile features - done
-- Check for working CXX compiler: /usr/bin/c++
-- Check for working CXX compiler: /usr/bin/c++ -- works
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Looking for pthread.h
-- Looking for pthread.h - found
-- Looking for pthread_create
-- Looking for pthread_create - not found
-- Looking for pthread_create in pthreads
-- Looking for pthread_create in pthreads - not found
-- Looking for pthread_create in pthread
-- Looking for pthread_create in pthread - found
-- Found Threads: TRUE
-- Configuring done
-- Generating done
```

I think the pthread_create here is a red herring for the error as the true error is as written below:

Issue I found was that hello_world example was not being linked against libvaccel and libdl... not sure why the makefile doesn't overcome this step but in essence:

```
gcc -I ../src/include/ -I ../third-party/slog/src ../helloworld.c
gcc -L src helloworld.o -o helloworld -lvaccel -ldl
```
fixed it for me at the moment, I will try and find a CMakefile fix as well.


**Lab3**:
Before starting, make sure:
```
$sudo apt update 
```
Steps are fine otherwise

Make sure the platform is the one you are looking for, an example:

```
Using platform: Portable Computing Language
Using device: pthread-cortex-a9
 result:
0 2 4 3 5 7 6 8 10 9
```


