# lab 5: writing a plugin/operation for an arbitary function

This is a rather long lab so it has been separated in small(er) sections for it to be easily digestable. 


# Creating an operation
Using the knowledge from the previous labs and with some tinkering with the source files, it is possible to add an operation to the vaccelrt files for whichever function you wish to decouple from the hardware.

Suppose we would like to create an operation called "FOO" and add it to the vaccelrt files:   

Working from the `src` directory:


Since we have an operation name in mind, lets add this first to the vaccelrt codebase.
In the `include/ops/vaccel_ops.h` file we can see there is an enum declration of `vaccel_op_type`:

Here there are some declarations made for some of the operations already used but we need to add our operation, "FOO" as shown below.

```C
enum vaccel_op_type {
	VACCEL_NO_OP = 0,
	VACCEL_BLAS_SGEMM,          /* 1 */
	VACCEL_IMG_CLASS,           /* 2 */
	VACCEL_IMG_DETEC,           /* 3 */
	VACCEL_IMG_SEGME,           /* 4 */
	VACCEL_IMG_POSE,            /* 5 */
	VACCEL_IMG_DEPTH,           /* 6 */
	VACCEL_EXEC,                /* 7 */
	VACCEL_TF_MODEL_NEW,        /* 8 */
	VACCEL_TF_MODEL_DESTROY,    /* 9 */
	VACCEL_TF_MODEL_REGISTER,   /* 10 */
	VACCEL_TF_MODEL_UNREGISTER, /* 11 */
	VACCEL_TF_SESSION_LOAD,     /* 12 */
	VACCEL_TF_SESSION_RUN,      /* 13 */
	VACCEL_TF_SESSION_DELETE,   /* 14 */
	VACCEL_FOO,				    /* 15 */
	VACCEL_FUNCTIONS_NR,		/* Insert operation ABOVE this */
};
```

Underneath in this same file, we have the operation name array ```vaccel_op_name``` and we can just add the description of the operation in the relevant position. In this case, we can add `foo` as the description.

Great, now we can save this file and return back `/include/ops` directory.

---
Now we create a header file named `foo.h` and for the purpose of this exercise, you can feel free to use the template below.

```C
#ifndef __VACCEL_FOO_H__
#define __VACCEL_FOO_H__

#include <stdint.h>
#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

struct vaccel_session;

int vaccel_foo(struct vaccel_session *sess, uint32_t a, uint32_t b, uint32_t c);

#ifdef __cplusplus
}
#endif

#endif /* __VACCEL_FOO_H__ */
```

One thing to note here is that we must declare what variables our operation is going to use; for the purpose of this, let us say that we are going to take 2 *READ* variables a and b and add them up and then add them together and *WRITE* them to variable c. Note the emphasis on *READ* and *WRITE* here, this will be important later on. So do note this down somewhere if your function is complex. Also the `struct vaccel_session *sess` is important here to init the vaccelrt session.

---

We can now return to directory `/include`.

The only change we need to do here is in the `vaccel.h` file. Here, we can add our header `#include "ops/foo.h"` in this file.

We can now leave the `include` directory completely and move onto the `src/ops` directory.

In this directory, as you may see, there will be several files for each operation with its relevant c file and then the header file to accompany with. So as before, lets create two files in this directory called `foo.h` and `foo.c`.

Working with the simpler one first, lets look at the header file: 

Here you can just borrow the template once again, noting that this time this will be an unpack function of the variables we mentioned in the previous part of the lab.

```C
#ifndef __FOO_H__
#define __FOO_H__

#include <stddef.h>
#include <stdint.h>

#include "include/ops/foo.h"

struct vaccel_session;
struct vaccel_arg;

int vaccel_foo_unpack(struct vaccel_session *sess, struct vaccel_arg *read,
		int nr_read, struct vaccel_arg *write, int nr_write);

#endif /* __FOO_H__ */

```

Now looking at the complementary `foo.c` file:

```C
#include "foo.h"
#include "error.h"
#include "plugin.h"
#include "log.h"
#include "vaccel_ops.h"
#include "genop.h"

#include "session.h"
int vaccel_foo(struct vaccel_session *sess, uint32_t a, uint32_t b, uint32_t c)
{
	if (!sess)
		return VACCEL_EINVAL;

	vaccel_debug("session:%u Looking for plugin implementing FOO operation",
			sess->session_id);

	//Get implementation
	int (*plugin_op)() = get_plugin_op(VACCEL_FOO);
	if (!plugin_op)
		return VACCEL_ENOTSUP;

	return plugin_op(sess, a, b, c);
}

int vaccel_foo_unpack(struct vaccel_session *sess, struct vaccel_arg *read,
		int nr_read, struct vaccel_arg *write, int nr_write)
{
	if (nr_read != 2) {
		vaccel_error("Wrong number of read arguments in FOO: %d",
				nr_read);
		return VACCEL_EINVAL;
	}

	if (nr_write != 1) {
		vaccel_error("Wrong number of write arguments in FOO: %d",
				nr_write);
		return VACCEL_EINVAL;
	}
	uint32_t a = *(uint32_t*)read[0].buf;
	uint32_t b = *(uint32_t*)read[1].buf;
	uint32_t c = *(uint32_t*)write[0].buf;

	return vaccel_foo(sess, a, b, c);
}
```
The first part of the file tries to find an implementation of the operation FOO in the plugin (which we will talk about later on) and if it does find an implementation - it returns it.

The latter part of the file unpacks the variables. Pay attention to the variables `nr_read` and `nr_write` and make sure they correspond to the correct number of read and write variables. In this example I just use 2, 1 as I only have 2 read variables and 1 write variable, as before and it is relatively simple to implement. For more complex examples, feel free to explore the other .c files in this directory to get an idea of the structure you must implement for this file.

Cool! Just a few more things in this directory before we can close it for good!:

---

Next in the `genop.c` file add the relevant `#include "foo.h"` header once more and include our operation in the `typedef structure`:

```C
unpack_func_t callbacks[VACCEL_FUNCTIONS_NR] = {
	vaccel_noop_unpack,
	vaccel_sgemm_unpack,
	vaccel_image_classification_unpack,
	vaccel_image_detection_unpack,
	vaccel_image_segmentation_unpack,
	vaccel_image_pose_unpack,
	vaccel_image_depth_unpack,
	vaccel_exec_unpack,
	vaccel_foo_unpack,
};
```

Add the `vaccel_foo_unpack` function into this array and we can now save and close this file. And thats it for this directoy! Lets move on to the overall `src` directory once more.

Final thing to note here is to once more add the `#include "ops/foo.h"` in the `vaccel.h` file and we are done!

Now lets make sure this operation works before we start working on the actual function itself.

---

## Testing the operation

We are just going to edit the noop plugin here as mentioned in the previous labs as most of the framework of the plugin is already written and it is much quicker to test it.
Would rather find out something is wrong instead of writing an entirely new plugin and trying to debug two things at once - not fun!

So, starting at the top of the `vaccelrt` directory, we need to go to `plugins/noop` directory and edit the `vaccel.c` file here:

Once again, add the `#include <ops/foo.h>` at the top of the file and now we can scroll down until we get to:

```C
struct vaccel_op ops[] = {
	VACCEL_OP_INIT(ops[0], VACCEL_NO_OP, noop_noop),
	VACCEL_OP_INIT(ops[1], VACCEL_BLAS_SGEMM, noop_sgemm),
	VACCEL_OP_INIT(ops[2], VACCEL_IMG_CLASS, noop_img_class),
	VACCEL_OP_INIT(ops[3], VACCEL_IMG_DETEC, noop_img_detect),
	VACCEL_OP_INIT(ops[4], VACCEL_IMG_SEGME, noop_img_segme),
	VACCEL_OP_INIT(ops[5], VACCEL_EXEC, noop_exec),
	VACCEL_OP_INIT(ops[6], VACCEL_TF_SESSION_LOAD, noop_tf_session_load),
	VACCEL_OP_INIT(ops[7], VACCEL_TF_SESSION_RUN, noop_tf_session_run),
	VACCEL_OP_INIT(ops[8], VACCEL_TF_SESSION_DELETE, noop_tf_session_delete),
	VACCEL_OP_INIT(ops[9], VACCEL_FOO, noop_foo),
};
```
Here we are initialising the operations for the vaccel runtime and we can just add an extra element into the array `ops[]` as seen above as:

```C
VACCEL_OP_INIT(ops[9], VACCEL_FOO, noop_foo)
```

`VACCEL_FOO` is the operation we are creating and we have a function using this operation which we will call `noop_foo`.

Now we have to actually implement this `noop_foo` function otherwise we will not be able be use this operation.

```C
static int noop_foo(struct vaccel_session *sess, uint32_t a, uint32_t b,
		      uint32_t c)
{
	fprintf(stdout, "[noop] Calling foo for session %u\n",
		sess->session_id);	
	return VACCEL_OK;
}
```
Paste this above the struct and it should be ready to go.
Great! Now just repeat the same exercise we did in lab1. To sum up the required commands:

```bash
cd ../..
mkdir -p build
cd build
cmake ../ -DBUILD_PLUGIN_NOOP=ON
make
gcc -I ../src/include/ -I ../third-party/slog/src ../examples/noop.c -c
gcc -L src noop.o -o noop -lvaccel -ldl
LD_LIBRARY_PATH=./src VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/noop/libvaccel-noop.so  ./noop
```

We can see in the debug console that the operation FOO should have been implementated:

```
2022.03.21-20:04:36.82 - <debug> Initializing vAccel
2022.03.21-20:04:36.82 - <debug> Registered plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function noop from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function sgemm from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function image classification from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function image detection from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function image segmentation from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function exec from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function TensorFlow session load from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function TensorFlow session run from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function TensorFlow session delete from plugin noop
2022.03.21-20:04:36.82 - <debug> Registered function Foo function from plugin noop
```

## Function implementation

Now for the function implementation, we can just write out our own plugin which will then contain the function as needed:


As we did in the third lab, create a new folder in the plugins directory and lets call it `foo_plugin`.

Once again for `CMakeLists.txt`:

```
set(include_dirs ${CMAKE_SOURCE_DIR}/src)
set(SOURCES vaccel.c ${include_dirs}/vaccel.h ${include_dirs}/plugin.h)

add_library(vaccel-foo SHARED ${SOURCES})
target_include_directories(vaccel-foo PRIVATE ${include_dirs})

# Setup make install
install(TARGETS vaccel-foo DESTINATION "${lib_path}")
```

And for the function implementation `vaccel.c`:
Lets say for the purpose of this plugin it takes 2 variables, `a` and `b` and adds them together and then prints out the result - rather basic but you get the idea of it :) Here is the static API implementation of the function:

```C
#include <stdio.h>
#include <plugin.h>

static int foo(struct vaccel_session *session, uint32_t a, uint32_t b, uint32_t c)
{
	fprintf(stdout, "Calling vaccel-foo for session %u\n", session->session_id);
	printf("---\n\n");

	printf("Calling foo operation here\n");
	c = a + b;
	printf("%d", c);

	printf("\n\n---\n\n");
	printf("Ending foo operation here");

	return VACCEL_OK;
}

struct vaccel_op op = VACCEL_OP_INIT(op, VACCEL_FOO, foo);

static int init(void)
{
	return register_plugin_function(&op);
}

static int fini(void)
{
	return VACCEL_OK;
}

VACCEL_MODULE(
	.name = "foo",
	.version = "0.1",
	.init = init,
	.fini = fini
)
```

Now lets just create a basic `foo_example.c` in the examples folder, just like the hello world example:

```C
#include <stdlib.h>
#include <stdio.h>

#include <vaccel.h>

int main()
{
	int ret;
	struct vaccel_session sess;
	
	uint32_t a = 5, b = 5, c = 0;

	ret = vaccel_sess_init(&sess, 0);
	if (ret != VACCEL_OK) {
		fprintf(stderr, "Could not initialize session\n");
		return 1;
	}


	printf("Initialized session with id: %u\n", sess.session_id);


	ret = vaccel_foo(&sess, a, b, c);
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

To build the plugin (as we did in lab2), add the following snippet to `plugins/CMakeLists.txt`:

```cmake
...
...

if (BUILD_PLUGIN_FOO)
        add_subdirectory(foo_plugin)
endif(BUILD_PLUGIN_FOO)
```
and to `CMakeLists.txt`:

```cmake
...
...

option(BUILD_PLUGIN_FOO "Build the foo plugin" OFF)
```

Now we can build our new plugin, by specifying the new option we just added:

```bash
cd build
cmake ../ -DBUILD_PLUGIN_FOO=ON
make
```

Now let's build the example:

```bash
gcc -I ../src/include/ -I ../third-party/slog/src ../examples/foo_example.c -c
gcc -L src foo_example.o -o foo_example -lvaccel -ldl
```

And run it:

```bash
LD_LIBRARY_PATH=./src VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./plugins/foo_plugin/libvaccel-foo.so  ./foo_example
```


And now for the result:

```
Initialized session with id: 1
Calling vaccel-foo for session 1
---

Calling foo operation here

10

---

Ending foo operation here

```

Perfect, works as intended.

---

Hopefully this would give you a rather good idea of how to implement your own function into vAccel and for further examples just have a quick glance through some of the more complex examples in the corresponding directories.

