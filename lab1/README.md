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
`VACCEL_LOG_LEVEL` environment variable, to shed a bit of light:

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu VACCEL_LOG_LEVEL=4  ./bin/noop
```
should return
```
2025.03.23-09:45:58.89 - <debug> Initializing vAccel
2025.03.23-09:45:58.89 - <info> vAccel 0.6.1-191-f1337abd
2025.03.23-09:45:58.89 - <debug> Config:
2025.03.23-09:45:58.89 - <debug>   plugins = (null)
2025.03.23-09:45:58.89 - <debug>   log_level = debug
2025.03.23-09:45:58.89 - <debug>   log_file = (null)
2025.03.23-09:45:58.89 - <debug>   profiling_enabled = false
2025.03.23-09:45:58.89 - <debug>   version_ignore = false
2025.03.23-09:45:58.89 - <debug> Created top-level rundir: /run/user/1008/vaccel/Kd6iwd
2025.03.23-09:45:58.89 - <debug> New rundir for session 1: /run/user/1008/vaccel/Kd6iwd/session.1
2025.03.23-09:45:58.89 - <debug> Initialized session 1
Initialized session with id: 1
2025.03.23-09:45:58.89 - <debug> session:1 Looking for plugin implementing noop
2025.03.23-09:45:58.89 - <warn> None of the loaded plugins implement VACCEL_OP_NOOP
Could not run op: 95
2025.03.23-09:45:58.89 - <debug> Released session 1
2025.03.23-09:45:58.89 - <debug> Cleaning up vAccel
2025.03.23-09:45:58.89 - <debug> Cleaning up sessions
2025.03.23-09:45:58.89 - <debug> Cleaning up resources
2025.03.23-09:45:58.89 - <debug> Cleaning up plugins
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
meson setup --buildtype=release build --prefix=<path> -Dplugin-noop=enabled --reconfigure
```

To run our example using the `noop` plugin, we need to set the environment
variable `VACCEL_PLUGINS`. When we execute the example and specify the
plugin under the `<path>` directory:

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu VACCEL_LOG_LEVEL=4 VACCEL_PLUGINS=./lib/x86_64-linux-gnu/libvaccel-noop.so ./bin/noop
```
we get the following output:

```
2025.03.23-09:46:52.12 - <debug> Initializing vAccel
2025.03.23-09:46:52.12 - <info> vAccel 0.6.1-191-f1337abd
2025.03.23-09:46:52.12 - <debug> Config:
2025.03.23-09:46:52.12 - <debug>   plugins = ./lib/x86_64-linux-gnu/libvaccel-noop.so
2025.03.23-09:46:52.12 - <debug>   log_level = debug
2025.03.23-09:46:52.12 - <debug>   log_file = (null)
2025.03.23-09:46:52.12 - <debug>   profiling_enabled = false
2025.03.23-09:46:52.12 - <debug>   version_ignore = false
2025.03.23-09:46:52.12 - <debug> Created top-level rundir: /run/user/1008/vaccel/ehxGl2
2025.03.23-09:46:52.12 - <info> Registered plugin noop 0.6.1-191-f1337abd
2025.03.23-09:46:52.12 - <debug> Registered op noop from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op blas_sgemm from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op image_classify from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op image_detect from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op image_segment from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op image_pose from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op image_depth from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op exec from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op tf_session_load from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op tf_session_run from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op tf_session_delete from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op minmax from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op fpga_arraycopy from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op fpga_vectoradd from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op fpga_parallel from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op fpga_mmult from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op exec_with_resource from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op torch_jitload_forward from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op torch_sgemm from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op opencv from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op tflite_session_load from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op tflite_session_run from plugin noop
2025.03.23-09:46:52.12 - <debug> Registered op tflite_session_delete from plugin noop
2025.03.23-09:46:52.12 - <debug> Loaded plugin noop from ./lib/x86_64-linux-gnu/libvaccel-noop.so
2025.03.23-09:46:52.12 - <debug> New rundir for session 1: /run/user/1008/vaccel/ehxGl2/session.1
2025.03.23-09:46:52.12 - <debug> Initialized session 1
Initialized session with id: 1
2025.03.23-09:46:52.12 - <debug> session:1 Looking for plugin implementing noop
2025.03.23-09:46:52.12 - <debug> Returning func from hint plugin noop
2025.03.23-09:46:52.12 - <debug> Found implementation in noop plugin
2025.03.23-09:46:52.12 - <debug> [noop] Calling no-op for session 1
2025.03.23-09:46:52.12 - <debug> Released session 1
2025.03.23-09:46:52.12 - <debug> Cleaning up vAccel
2025.03.23-09:46:52.12 - <debug> Cleaning up sessions
2025.03.23-09:46:52.12 - <debug> Cleaning up resources
2025.03.23-09:46:52.12 - <debug> Cleaning up plugins
2025.03.23-09:46:52.12 - <debug> Unregistered plugin noop
```

## Takeaway

We saw how we build the vAccel runtime system. We then learnt what a vAccel
Hello, World application looks like and how we build it. Finally, we saw how
we build one of the available vAccel runtime plugins and use it to execute our
application.
