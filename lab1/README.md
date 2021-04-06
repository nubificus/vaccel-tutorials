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

and execute the noop example which essentially does the following:
```
#include <stdlib.h>
#include <stdio.h>

#include <vaccel.h>
#include <vaccel_ops.h>

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
	if (ret) {
		fprintf(stderr, "Could not run op: %d\n", ret);
		goto close_session;
	}

close_session:
	if (vaccel_sess_free(&sess) != VACCEL_OK) {
		fprintf(stderr, "Could not clear session\n");
		return 1;
	}

	return ret;
}
```

The output is a bit confusing at first:

```
$ ./examples/noop 
Initialized session with id: 1
Could not run op: 95
```

Enabling debug messages sheds a bit of light:

```
$ VACCEL_DEBUG_LEVEL=4 ./examples/noop 
2021.04.01-17:19:06.91 - <debug> Initializing vAccel
2021.04.01-17:19:06.91 - <debug> session:1 New session
Initialized session with id: 1
2021.04.01-17:19:06.91 - <debug> session:1 Looking for plugin implementing noop
2021.04.01-17:19:06.91 - <warn> None of the loaded plugins implement noop
Could not run op: 95
2021.04.01-17:19:06.91 - <debug> session:1 Free session
2021.04.01-17:19:06.91 - <debug> Shutting down vAccel
2021.04.01-17:19:06.91 - <debug> Cleaning up plugins
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

and when we execute the example, we specify which plugin to use:

```
VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/noop/libvaccel-noop.so ./examples/noop
```

Now the output seems a bit more clear:

```
2021.04.01-17:23:23.68 - <debug> Initializing vAccel
2021.04.01-17:23:23.68 - <debug> Registered plugin noop
2021.04.01-17:23:23.68 - <debug> Registered function noop from plugin noop
2021.04.01-17:23:23.68 - <debug> Loaded plugin noop from ./plugins/noop/libvaccel-noop.so
2021.04.01-17:23:23.68 - <debug> session:1 New session
Initialized session with id: 1
2021.04.01-17:23:23.68 - <debug> session:1 Looking for plugin implementing noop
2021.04.01-17:23:23.68 - <debug> Found implementation in noop plugin
Calling no-op for session 1
2021.04.01-17:23:23.68 - <debug> session:1 Free session
2021.04.01-17:23:23.68 - <debug> Shutting down vAccel
2021.04.01-17:23:23.68 - <debug> Cleaning up plugins
2021.04.01-17:23:23.68 - <debug> Unregistered plugin noop
```

### Takeaway

We have built the core runtime system, and we used the noop example to see what
happens when no plugins are included. We then built a plugin that implements
this operation (`noop`) and saw that the function succeeds when a relevant
implementation of the operation exists.
