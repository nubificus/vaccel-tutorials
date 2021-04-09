# lab2: how to write a vAccel plugin

In [lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md)
we saw how to write, build and execute a simple vAccel application calling the
`vaccel_noop` function which is implemented by the `noop` plugin.

In this lab, we will go through the process of writing our own vAccel plugin
that implements the `vaccel_noop` that we used in our Hello, World example in
[lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md).
The purpose of this exercise is to showcase the rationale of the plugin/backend
system and the simplicity of adding a new plugin to vAccel.

To implement a vAccel plugin, all we need to do is add the implementation of
the relevant frontend function and register it to the vAccel plugin subsystem.

To start, we assume we have built and familiarized ourselves with vAccelRT
using
[lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md).


We will use the `noop` plugin as a skeleton for our new plugin which we will
call `helloworld`. For simplicity we copy the `noop` plugin directory structure
to a new directory:

```
cp -avf plugins/noop plugins/helloworld
```

we replace `noop` with the name of our choosing (`helloworld`) so we come up with
the following directory structure:

```
plugins/helloworld
├── CMakeLists.txt
└── vaccel.c
```

We need to change the contents of `plugins/helloworld/CMakeLists.txt` to reflect
the `noop` to `helloworld` change:

```cmake
set(include_dirs ${CMAKE_SOURCE_DIR}/src)
set(SOURCES vaccel.c ${include_dirs}/vaccel.h ${include_dirs}/plugin.h)

add_library(vaccel-helloworld SHARED ${SOURCES})
target_include_directories(vaccel-helloworld PRIVATE ${include_dirs})

target_link_libraries(vaccel-helloworld PRIVATE vector_add OpenCL)

# Setup make install
install(TARGETS vaccel-helloworld DESTINATION "${lib_path}")
```

Similarly, we replace the plugin implementation with our own in `vaccel.c`,
adding a simple message:

```C
#include <stdio.h>
#include <plugin.h>

static int helloworld(struct vaccel_session *session)
{
	fprintf(stdout, "Calling vaccel-helloworld for session %u\n", session->session_id);

	printf("_______________________________________________________________\n\n");
	printf("This is the helloworld plugin, implementing the NOOP operation!\n");
	printf("===============================================================\n\n");

	return VACCEL_OK;
}

struct vaccel_op op = VACCEL_OP_INIT(op, VACCEL_NO_OP, helloworld);

static int init(void)
{
	return register_plugin_function(&op);
}

static int fini(void)
{
	return VACCEL_OK;
}

VACCEL_MODULE(
	.name = "helloworld",
	.version = "0.1",
	.init = init,
	.fini = fini
)
```

Additionally, we need to add the relevant rules in the `CMakeLists.txt` files
to build the plugin along with the rest of the vAccelRT code.

We add the following to `plugins/CMakeLists.txt`:

```cmake
if (BUILD_PLUGIN_HELLOWORLD)
        add_subdirectory(helloworld)
endif(BUILD_PLUGIN_HELLOWORLD)
```

and to `CMakeLists.txt`:

```cmake
option(BUILD_PLUGIN_HELLOWORLD "Build the hello-world debugging plugin" OFF)
```

Now we can build our new plugin, by specifying the new option we just added:

```sh
cd build
cmake ../ -DBUILD_PLUGIN_HELLOWORLD=ON
make
```

We see that a new plugin is now available, `libvaccel-helloworld.so`. Lets use this
one instead of the `noop` one!

```sh
$ LD_LIBRARY_PATH=src VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/helloworld/libvaccel-helloworld.so ./hello_world
2021.04.09-12:40:45.86 - <debug> Initializing vAccel
2021.04.09-12:40:45.86 - <debug> Registered plugin helloworld
2021.04.09-12:40:45.86 - <debug> Registered function noop from plugin helloworld
2021.04.09-12:40:45.86 - <debug> Loaded plugin helloworld from ./plugins/helloworld/libvaccel-helloworld.so
2021.04.09-12:40:45.86 - <debug> session:1 New session
Initialized session with id: 1
2021.04.09-12:40:45.86 - <debug> session:1 Looking for plugin implementing noop
2021.04.09-12:40:45.86 - <debug> Found implementation in helloworld plugin
Calling vaccel-helloworld for session 1
_______________________________________________________________

This is the helloworld plugin, implementing the NOOP operation!
===============================================================

2021.04.09-12:40:45.86 - <debug> session:1 Free session
2021.04.09-12:40:45.86 - <debug> Shutting down vAccel
2021.04.09-12:40:45.86 - <debug> Cleaning up plugins
2021.04.09-12:40:45.86 - <debug> Unregistered plugin helloworld
```

### Takeaway

We made it! We just added our own plugin to implement a vAccel operation: 
- we integrated it into the vAccel source tree, 
- we built it and 
- used it to run the `noop` operation.
