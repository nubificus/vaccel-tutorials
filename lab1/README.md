# lab1: build vAccel & run a simple operation

In this short tutorial, we will go through the build process of the vAccel
runtime and how you can run the `hello-world` vAccel application using one
of the available back-end plugins.

## Building vAccel

### Installing the core runtime

Build and install the core runtime library from the latest release of the [vAccel](https://github.com/nubificus/vaccel) repository (steps 1 and 2).

Use the `--prefix=<path>` option in the meson setup command to specify a particular installation path.

Meson setup returns the following configuration info which shows that none of the plugins are enabled. In the next step will see how to enable plugins and run a simple application.
```
...
 Configuration
    Build the exec plugin           : NO
    Build the no-op debugging plugin: NO
    Build the mbench plugin         : NO
    Build the examples              : NO
    Enable testing                  : NO
...
```

## vAccel Hello World!

The `Hello, world!` example of vAccel is a simple application that calls the
`vaccel_noop` function of the [vAccel API](https://docs.vaccel.org/api).

As you have probably guessed the `vaccel_noop` function is a no-op, it does not
require any actual acceleration, but it's useful for demonstration and debugging
purposes, so we will be using it along the course of this and following labs.

You can find the code [here](https://github.com/nubificus/vaccel/blob/main/src/ops/noop.c). The `vaccel_noop` function calls the implementation the `noop_noop` function as implemented by the no-op plugin.

All vAccel operations are performed in the context of a "user" session, so the
first thing that the program does is creating a new `vaccel_session`.
Next, it performs the actual operation `vaccel_noop` and closes the session.

### Building plugin and application

To run the hello world example, we need first to build the no-op plugin. The `-Dexamples=enabled` option build the available vAccel example. The `-Dplugin-noop=enabled` option build the no-op plugin.
```
meson setup --buildtype=release build --prefix=<path> -Dexamples=enabled -Dplugin-noop=enabled --reconfigure
```
Or:
```
meson setup --buildtype=release build --prefix=<path> -Dauto_features=enabled -Dplugin-noop=enabled --reconfigure
```

To view all available options and their values, you can use:
```
meson configure build
```

Then, compile and install the project after the changes.
```
meson compile -C build
meson install -C build
```

Now, you should find all the vAccel examples, including `noop`  under the `<path>/bin` directory.

### Running the application

Let's move to the `<path>` directory to run our 'Hello, World' application:

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu ./bin/noop
```
should return
```
Initialized session with id: 1
Could not run op: 95
```

Not what we expected. Let's enable vAccel runtime debugging, by setting the
`VACCEL_DEBUG_LEVEL` environment variable, to shed a bit of light:

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu VACCEL_DEBUG_LEVEL=4  ./bin/noop
```
should return
```
2025.01.07-11:58:35.17 - <debug> Initializing vAccel
2025.01.07-11:58:35.17 - <info> vAccel 0.6.1-164-dcf00bfd
2025.01.07-11:58:35.17 - <debug> Created top-level rundir: /run/user/1009/vaccel/2y58Fn
2025.01.07-11:58:35.17 - <debug> New rundir for session 1: /run/user/1009/vaccel/2y58Fn/session.1
2025.01.07-11:58:35.17 - <debug> Initialized session 1
Initialized session with id: 1
2025.01.07-11:58:35.17 - <debug> session:1 Looking for plugin implementing noop
2025.01.07-11:58:35.17 - <warn> None of the loaded plugins implement noop
Could not run op: 95
2025.01.07-11:58:35.17 - <debug> Released session 1
2025.01.07-11:58:35.17 - <debug> Shutting down vAccel
2025.01.07-11:58:35.17 - <debug> Cleaning up plugins
```

Ok better!

What we see first here is the vAccel runtime initializing. This happens
automatically during application loading. Next, we see the successful
creation of a new vAccel session with id `1`. Finally, upon call of
`vaccel_noop` from our program, the runtime looks for a plugin which implements
the operation and is unable to find one, so the call fails. Finally, the
vAccel session `1` is freed and the execution exits.

### Build the `noop` plugin

Let's go back and build a plugin that implements this operation, `noop`:

```
meson setup --buildtype=release build --prefix=$HOME/binaries/test/vaccel -Dplugin-noop=enabled --reconfigure
```

To run our example using the `noop` plugin, we need to set the environment
variable `VACCEL_BACKENDS`. When we execute the example and specify the
plugin under the `<path>` directory:

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./lib/x86_64-linux-gnu/libvaccel-noop.so ./bin/noop
```
we get the following output:

```
2025.01.07-11:59:52.72 - <debug> Initializing vAccel
2025.01.07-11:59:52.72 - <info> vAccel 0.6.1-166-3cfa48b7
2025.01.07-11:59:52.72 - <debug> Created top-level rundir: /run/user/1009/vaccel/4pv5d6
2025.01.07-11:59:52.72 - <info> Registered plugin noop 0.6.1-166-3cfa48b7
2025.01.07-11:59:52.72 - <debug> Registered function noop from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function sgemm from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function image classification from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function image detection from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function image segmentation from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function image pose estimation from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function image depth estimation from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function exec from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function TensorFlow session load from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function TensorFlow session run from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function TensorFlow session delete from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function MinMax from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function Array copy from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function Vector Add from plugin
noop
2025.01.07-11:59:52.72 - <debug> Registered function Parallel acceleration from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function Matrix multiplication from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function Exec with resource from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function Torch jitload_forward function from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function Torch SGEMM from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function OpenCV Generic from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function TensorFlow Lite session load from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function TensorFlow Lite session run from plugin noop
2025.01.07-11:59:52.72 - <debug> Registered function TensorFlow Lite session delete from plugin noop
2025.01.07-11:59:52.72 - <debug> Loaded plugin noop from ./lib/x86_64-linux-gnu/libvaccel-noop.so
2025.01.07-11:59:52.72 - <debug> New rundir for session 1: /run/user/1009/vaccel/4pv5d6/session.1
2025.01.07-11:59:52.72 - <debug> Initialized session 1
Initialized session with id: 1
2025.01.07-11:59:52.72 - <debug> session:1 Looking for plugin implementing noop
2025.01.07-11:59:52.72 - <debug> Returning func from hint plugin noop
2025.01.07-11:59:52.72 - <debug> Found implementation in noop plugin
2025.01.07-11:59:52.72 - <debug> [noop] Calling no-op for session 1
2025.01.07-11:59:52.72 - <debug> Released session 1
2025.01.07-11:59:52.72 - <debug> Shutting down vAccel
2025.01.07-11:59:52.72 - <debug> Cleaning up plugins
2025.01.07-11:59:52.72 - <debug> Unregistered plugin noop
```

## Takeaway

We saw how we build the vAccel runtime system. We then learnt what a vAccel
Hello, World application looks like and how we build it. Finally, we saw how
we build one of the available vAccel runtime plugins and use it to execute our
application.
