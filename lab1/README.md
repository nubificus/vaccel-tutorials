# lab1: build vAccel & run a simple operation

In this short tutorial, we will go through the build process of vAccel and 
familiarize ourselves with running a simple operation. 

## Building vAccelRT

### Get the code

Clone the vAcceRT repository using the following command:

```
git clone https://github.com/cloudkernels/vaccelrt
```

Enter the path and fetch all submodules:

```
cd vaccelrt
git submodule update --init
```

### Build the core runtime

Create a build directory and prepare the relevant files for building (including
just the examples):

```
mkdir build
cd build
cmake ../ -DBUILD_EXAMPLES=ON
```

Build the source:

```
make
```

### Execute `noop`

and execute the [noop example](https://github.com/cloudkernels/vaccelrt/blob/master/examples/noop.c) which essentially does the following:
```
	ret = vaccel_sess_init(&sess, 0);
	ret = vaccel_noop(&sess);
	ret = vaccel_sess_free(&sess);
```

The output is a bit confusing at first:

```
$ ./examples/noop 
Initialized session with id: 1
Could not run op: 95
```

To enable debug, we need to set the environment variable `VACCEL_DEBUG_LEVEL`.
Enabling debug messages sheds a bit of light:

```
$ VACCEL_DEBUG_LEVEL=4 ./examples/noop 
2021.04.09-09:09:03.39 - <debug> Initializing vAccel
2021.04.09-09:09:03.39 - <debug> session:1 New session
Initialized session with id: 1
2021.04.09-09:09:03.39 - <debug> session:1 Looking for plugin implementing noop
2021.04.09-09:09:03.39 - <warn> None of the loaded plugins implement noop
Could not run op: 95
2021.04.09-09:09:03.39 - <debug> session:1 Free session
2021.04.09-09:09:03.39 - <debug> Shutting down vAccel
2021.04.09-09:09:03.39 - <debug> Cleaning up plugins
```

We can see that initialization works, the vAccel session is created but the
system fails to find a relevant plugin that implements the operation we asked
for: `noop`. 

### Build a `noop` plugin

Let's go back and build a plugin that implements this operation, `noop`:

```
cmake ../ -DBUILD_EXAMPLES=ON -DBUILD_PLUGIN_NOOP=ON
make
```

To run our example using the `noop` plugin, we need to set the environment
variable `VACCEL_BACKENDS`. When we execute the example and specify the
plugin to use we get the following:

```
$ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/noop/libvaccel-noop.so ./examples/noop
2021.04.09-09:10:03.48 - <debug> Initializing vAccel
2021.04.09-09:10:03.48 - <debug> Registered plugin noop
2021.04.09-09:10:03.48 - <debug> Registered function noop from plugin noop
2021.04.09-09:10:03.48 - <debug> Loaded plugin noop from ./plugins/noop/libvaccel-noop.so
2021.04.09-09:10:03.48 - <debug> session:1 New session
Initialized session with id: 1
2021.04.09-09:10:03.48 - <debug> session:1 Looking for plugin implementing noop
2021.04.09-09:10:03.48 - <debug> Found implementation in noop plugin
Calling no-op for session 1
2021.04.09-09:10:03.48 - <debug> session:1 Free session
2021.04.09-09:10:03.48 - <debug> Shutting down vAccel
2021.04.09-09:10:03.48 - <debug> Cleaning up plugins
2021.04.09-09:10:03.48 - <debug> Unregistered plugin noop
```

### Takeaway

We have built the core runtime system, and we used the noop example to see what
happens when no plugins are included. We then built a plugin that implements
this operation (`noop`) and saw that the function succeeds when a relevant
implementation of the operation exists.
