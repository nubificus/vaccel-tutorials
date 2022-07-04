
****FPGA setup:****

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

*For initial compilation:*
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



**Lab3**:
Before starting, make sure to update your Linux repositories list by using sudo apt update or the respective comamnd for your Linux build. If using FPGA-SoC-Linux as noted in the repository below, it may be well needed that you need a platform for the example in this lab to work so:

```
sudo apt update
sudo apt install pocl-opencl-icd
```
was the fix for the PYNQ-Z1.


Make sure the platform is the one you are looking for, an example:

```
Using platform: Portable Computing Language
Using device: pthread-cortex-a9
 result:
0 2 4 3 5 7 6 8 10 9
```

**Running sample tests on FPGA platform**

Using the samples in the repository:

https://github.com/ikwzm/FPGA-SoC-Linux

We can test if the device tree configuration system is working:

Just to test with, making sure the read and write module is active: (see repository for details);


```
cd build
cmake ../ -DBUILD_FPGA_PLUGIN=ON
make
```
Creating the shared object first:
```
gcc -I ../src/include/ -I ../third-party/slog/src ../examples/fpga_sample.c -c
gcc -L src fpga_sample.o -o fpga_sample -lvaccel -ldl
```
And then enabling vAccel runtime as usual:
```
LD_LIBRARY_PATH=./src/ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/fpga/libvaccel-fpga.so ./fpga_sample
```
and should give you a return value of:
```
2022.03.21-07:57:37.03 - <debug> Initializing vAccel
2022.03.21-07:57:37.03 - <debug> Registered plugin fpga
2022.03.21-07:57:37.03 - <debug> Registered function noop from plugin fpga
2022.03.21-07:57:37.03 - <debug> Loaded plugin fpga from ./plugins/fpga/libvaccel-fpga.so
2022.03.21-07:57:37.03 - <debug> session:1 New session
Initialized session with id: 1
2022.03.21-07:57:37.03 - <debug> session:1 Looking for plugin implementing noop
2022.03.21-07:57:37.03 - <debug> Found implementation in fpga plugin
Calling vaccel-fpga for session 1
_______________________________________________________________

This is the fpga plugin, implementing the NOOP operation!
===============================================================

2022.03.21-07:57:37.04 - <debug> session:1 Free session
------------------Sample 1 Start------------------
time = 0.006357 sec
time = 0.006239 sec
time = 0.006022 sec
time = 0.006035 sec
time = 0.006025 sec
time = 0.006021 sec
time = 0.006032 sec
time = 0.006002 sec
time = 0.006041 sec
time = 0.006036 sec
------------------Sample 1 Over------------------
------------------Sample 2 Start------------------
time = 0.006059 sec
time = 0.006052 sec
time = 0.006682 sec
time = 0.006400 sec
time = 0.006053 sec
time = 0.006159 sec
time = 0.005988 sec
time = 0.006056 sec
time = 0.006176 sec
time = 0.006070 sec
------------------Sample 2 End------------------
2022.03.21-07:57:38.66 - <debug> Shutting down vAccel
2022.03.21-07:57:38.66 - <debug> Cleaning up plugins
2022.03.21-07:57:38.66 - <debug> Unregistered plugin fpga
```


