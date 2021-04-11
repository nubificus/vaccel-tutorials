# lab1: build vAccel & run a simple operation

In this short tutorial, we will go through the build process of the vAccel
runtime and how you can run the `hello-world` vAccel application using one
of the available back-end plugins.

## Building vAccelRT

### Get the code

Clone the vAcceRT repository using the following command:

```
git clone --recursive https://github.com/cloudkernels/vaccelrt
cd vaccelrt
```

### Build the core runtime

Create a build directory and prepare the relevant files for building (including
just the examples):

```
mkdir build
cd build
cmake ../
```

Build the source:

```
make
```

You should find the vAccel runtime library under `src`

```
$ tree -L 1 src/
src/
├── CMakeFiles
├── cmake_install.cmake
├── libvaccel.so
├── Makefile
└── vaccel.pc

```

## vAccel Hello World!

The `Hello, world!` example of vAccel is a simple application that calls the
`vaccel_noop` function of the [vAccel API](https://docs.vaccel.org/devguide/api).

As you have probably guessed the `vaccel_noop` function is a no-op, it does not
require any actual acecleration, but it's useful for demonstration and debugging
purposes, so we will be using it along the course of this and following labs.

Here is the code which you can also find in the
[noop](https://github.com/cloudkernels/vaccelrt/blob/master/examples/noop.c)
example of the vAccelRT repo:

```C
#include <stdlib.h>
#include <stdio.h>

#include <vaccel.h>

int main()
{
	int ret;
	struct vaccel_session sess;

	ret = vaccel_sess_init(&sess, 0);
	if (ret != VACCEL_OK) {
		fprintf(stderr, "Could not initialize session\n");
		return 1;
	}

	printf("Initialized session with id: %u\n", sess.session_id);

	ret = vaccel_noop(&sess);
	if (ret)
		fprintf(stderr, "Could not run op: %d\n", ret);

	if (vaccel_sess_free(&sess) != VACCEL_OK) {
		fprintf(stderr, "Could not clear session\n");
		return 1;
	}

	return ret;
}
```

All vAccel operations are performed in the context of a "user" session, so the
first thing that the program does is creating a new `vaccel_session`.
Next, it performs the actual operation `vaccel_noop` and closes the session.

### Building an application

In order to build our "Hello, world" example we need to link it against
`libvaccel` which depends on `libdl`.

We put the above snippet in a file named `hello_world.c`, compile it and link
it.

```
gcc -I ../src hello_world.c -c
gcc -L src hello_world.o -o hello_world -lvaccel -ldl
```

The `-I` flag above tells gcc to look for vaccel.h header file under the src
directory of vaccelrt, whereas the `-L` flag it tells it to look for
`libvaccel.so` under `src` in the build directory.

### Running the application

Let's run our 'Hello, World' application:

```
$ LD_LIBRARY_PATH=./src ./hello_world
Initialized session with id: 1
Could not run op: 95
```

Not what we expected. Let's enable vAccel runtime debugging, by setting the
`VACCEL_DEBUG_LEVEL` environment variable, to shed a bit of light:

```
$ VACCEL_DEBUG_LEVEL=4 ./hello_world
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
cmake ../ -DBUILD_PLUGIN_NOOP=ON
make
```

To run our example using the `noop` plugin, we need to set the environment
variable `VACCEL_BACKENDS`. When we execute the example and specify the
plugin to use we get the following:

```
$ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/noop/libvaccel-noop.so ./hello_world
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

## Takeaway

We saw how we build the vAccel runtime system. We then learnt what a vAccel
Hello, World application looks like and how we build it. Finally, we saw how
he build one of the available vAccel runtime plugins and use it to execute our
application.
