# Add a plugin

In this short tutorial, we will go through adding a simple plugin to vAccel
that implements the `noop` operation we saw at
[lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md).

The purpose of this tutorial is to showcase the rationale of the plugin/backend
system and the simplicity of adding a new plugin to vAccel.

To implement a vAccel plugin, all we need to do is add the implementation of
the relevant frontend function and register it to the vAccel plugin subsystem.
We can use the `noop` plugin as reference.

To start, we assume we have built and familiarized ourselves with vAccelRT
using
[lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md).
Then, we go back to the vAccelRT base source dir and copy the `noop` plugin dir
contents to a new directory, called `mynoop`.

```
cp -avf plugins/noop plugins/mynoop
```

we replace `noop` with the name of our choosing (`mynoop`) so we come up with
the following directory structure:

```
plugins/mynoop
├── CMakeLists.txt
└── vaccel.c
```

We need to change the contents of `plugins/mynoop/CMakeLists.txt` to reflect
the `noop` to `mynoop` change:

```
set(include_dirs ${CMAKE_SOURCE_DIR}/src)
set(SOURCES vaccel.c ${include_dirs}/vaccel.h ${include_dirs}/plugin.h)

add_library(vaccel-mynoop SHARED ${SOURCES})
target_include_directories(vaccel-mynoop PRIVATE ${include_dirs})

target_link_libraries(vaccel-mynoop PRIVATE vector_add OpenCL)

# Setup make install
install(TARGETS vaccel-mynoop DESTINATION "${lib_path}")
```

Similarly, we replace the plugin implementation with our own in `vaccel.c`,
adding a simple message:

```
#include <stdio.h>
#include <plugin.h>

static int mynoop(struct vaccel_session *session)
{
	fprintf(stdout, "Calling myno-op for session %u\n", session->session_id);

	printf("_________________________________________________________\n\n");
	printf("This is the mynoop plugin, implementing the NOOP operation!\n");
	printf("=========================================================\n\n");

	return VACCEL_OK;
}

struct vaccel_op op = VACCEL_OP_INIT(op, VACCEL_NO_OP, mynoop);

static int init(void)
{
	return register_plugin_function(&op);
}

static int fini(void)
{
	return VACCEL_OK;
}

VACCEL_MODULE(
	.name = "mynoop",
	.version = "0.1",
	.init = init,
	.fini = fini
)
```

Additionally, we need to add the relevant rules in the `CMakeLists.txt` files
to build the plugin along with the rest of the vAccelRT code.

We add the following to `plugins/CMakeLists.txt`:

```
if (BUILD_PLUGIN_MYNOOP)
        add_subdirectory(mynoop)
endif(BUILD_PLUGIN_MYNOOP)
```

and to `CMakeLists.txt`:

```
option(BUILD_PLUGIN_MYNOOP "Build the mynoop debugging plugin" OFF)
```

Now we can build our new plugin, by specifying the new option we just added:

```
cmake ../ -DBUILD_EXAMPLES=ON -DBUILD_PLUGIN_MYNOOP=ON
make
```

We see that a new plugin is now available, `libvaccel-mynoop.so`. Lets use this
one instead of the `noop` one!

```
$ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/mynoop/libvaccel-mynoop.so ./examples/noop
2021.04.09-09:27:42.33 - <debug> Initializing vAccel
2021.04.09-09:27:42.33 - <debug> Registered plugin noop
2021.04.09-09:27:42.33 - <debug> Registered function noop from plugin noop
2021.04.09-09:27:42.33 - <debug> Loaded plugin noop from ./plugins/mynoop/libvaccel-mynoop.so
2021.04.09-09:27:42.33 - <debug> session:1 New session
Initialized session with id: 1
2021.04.09-09:27:42.33 - <debug> session:1 Looking for plugin implementing noop
2021.04.09-09:27:42.33 - <debug> Found implementation in noop plugin
Calling myno-op for session 1
_________________________________________________________

This is the mynoop plugin, implementing the NOOP operation!
=========================================================

2021.04.09-09:27:42.33 - <debug> session:1 Free session
2021.04.09-09:27:42.33 - <debug> Shutting down vAccel
2021.04.09-09:27:42.33 - <debug> Cleaning up plugins
2021.04.09-09:27:42.33 - <debug> Unregistered plugin noop
```

### Takeaway

We made it! We just added our own plugin to implement a vAccel operation: 
- we integrated it into the vAccel source tree, 
- we built it and 
- used it to run the `noop` operation.
