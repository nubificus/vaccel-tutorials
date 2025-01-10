# lab 5: writing a plugin/operation for an arbitary function

This is a rather long lab so it has been separated in small(er) sections for it to be easily digestable. 


# Creating an operation
Using the knowledge from the previous labs and with some tinkering with the source files, it is possible to add an operation to the vaccelrt files for whichever function you wish to decouple from the hardware.

Suppose we would like to create an operation called "FOO" and add it to the vaccelrt files:   

Working from the `vaccel/src` directory:


Since we have an operation name in mind, lets add this first to the vaccel codebase.
In the `include/vaccel/op.h` file we can see there is an enum declration of `vaccel_op_type`:

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
        VACCEL_MINMAX,              /* 15 */
        VACCEL_F_ARRAYCOPY,         /* 16 */
        VACCEL_F_MMULT,             /* 17 */
        VACCEL_F_PARALLEL,          /* 18 */
        VACCEL_F_VECTORADD,         /* 19 */
        VACCEL_EXEC_WITH_RESOURCE,  /* 20 */
        VACCEL_TORCH_JITLOAD_FORWARD, /* 21 */
        VACCEL_TORCH_SGEMM, /* 22 */
        VACCEL_OPENCV,  /* 23 */
        VACCEL_TFLITE_SESSION_LOAD,  /* 24 */
        VACCEL_TFLITE_SESSION_RUN,   /* 25 */
        VACCEL_TFLITE_SESSION_DELETE,/* 26 */
        VACCEL_FOO,  /* 27 */
        VACCEL_FUNCTIONS_NR
};
```

Underneath in this same file, we have the operation name array ```vaccel_op_name``` and we can just add the description of the operation in the relevant position. In this case, we can add `foo` as the description.

```C
static const char *vaccel_op_name[] = {
        "noop",
        "sgemm",
        "image classification",
        "image detection",
        "image segmentation",
        "image pose estimation",
        "image depth estimation",
        "exec",
        "TensorFlow model create",
        "TensorFlow model destroy",
        "TensorFlow model register",
        "TensorFlow model unregister",
        "TensorFlow session load",
        "TensorFlow session run",
        "TensorFlow session delete",
        "MinMax",
        "Array copy",
        "Matrix multiplication",
        "Parallel acceleration",
        "Vector Add",
        "Exec with resource",
        "Torch jitload_forward function",
        "Torch SGEMM",
        "OpenCV Generic",
        "TensorFlow Lite session load",
        "TensorFlow Lite session run",
        "TensorFlow Lite session delete",
        "foo",
        "Functions NR",
};
```

Great, now we can save this file and return to `/src/include/vaccel/ops` directory.

---
Now we create a header file named `foo.h` and for the purpose of this exercise, you can feel free to use the template below.

```C
#pragma once
#include "vaccel/session.h"

#ifdef __cplusplus
extern "C" {
#endif

int vaccel_foo(struct vaccel_session *sess, uint32_t a, uint32_t b, uint32_t c);

#ifdef __cplusplus
}
#endif
```

One thing to note here is that we must declare what variables our operation is going to use; for the purpose of this, let us say that we are going to take 2 *READ* variables a and b and add them up and then add them together and *WRITE* them to variable c. Note the emphasis on *READ* and *WRITE* here, this will be important later on. So do note this down somewhere if your function is complex.

---

We can now return to directory `/src/include`.

The only change we need to do here is in the `vaccel.h.in` file. Here, we can add our header `#include "vaccel/ops/foo.h"` in this file.

We can now leave the `include` directory completely and move onto the `src/ops` directory.

In this directory, as you may see, there will be several files for each operation with its relevant c file and then the header file to accompany with. So as before, lets create two files in this directory called `foo.h` and `foo.c`.

Working with the simpler one first, lets look at the header file: 

Here you can just borrow the template once again, noting that this time this will be an unpack function of the variables we mentioned in the previous part of the lab.

```C
#pragma once

#include "include/vaccel/ops/foo.h"
#include "arg.h"
#include "session.h"

#ifdef __cplusplus
extern "C" {
#endif

int vaccel_foo_unpack(struct vaccel_session *sess, struct vaccel_arg *read,
		int nr_read, struct vaccel_arg *write, int nr_write);

#ifdef __cplusplus
}
#endif
```

Now looking at the complementary `foo.c` file:

```C
#include "foo.h"
#include "error.h"
#include "plugin.h"
#include "log.h"
#include "op.h"
#include "genop.h"
#include "prof.h"
#include "session.h"

struct vaccel_prof_region foo_op_stats =
        VACCEL_PROF_REGION_INIT("vaccel_foo_op");

int vaccel_foo(struct vaccel_session *sess, uint32_t a, uint32_t b, uint32_t c)
{
	int ret;

	if (!sess)
		return VACCEL_EINVAL;

	vaccel_debug("session:%u Looking for plugin implementing FOO operation",
			sess->id);

	vaccel_prof_region_start(&foo_op_stats);

	//Get implementation
	int (*plugin_op)() = get_plugin_op(VACCEL_FOO, sess->hint);
	if (!plugin_op)
		return VACCEL_ENOTSUP;

	ret = plugin_op(sess, a, b, c);

	vaccel_prof_region_stop(&foo_op_stats);

	return ret;
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
        vaccel_noop_unpack, /* 0 */
        vaccel_sgemm_unpack, /* 1 */
        vaccel_image_classification_unpack, /* 2 */
        vaccel_image_detection_unpack, /* 3 */
        vaccel_image_segmentation_unpack, /* 4 */
        vaccel_image_pose_unpack, /* 5 */
        vaccel_image_depth_unpack, /* 6 */
        vaccel_exec_unpack, /* 7 */
        vaccel_noop_unpack, /* 8 */
        vaccel_noop_unpack, /* 9 */
        vaccel_noop_unpack, /* 10 */
        vaccel_noop_unpack, /* 11 */
        vaccel_noop_unpack, /* 12 */
        vaccel_noop_unpack, /* 13 */
        vaccel_noop_unpack, /* 14 */
        vaccel_minmax_unpack, /* 15 */
        vaccel_fpga_arraycopy_unpack, /* 16 */
        vaccel_fpga_mmult_unpack, /* 17 */
        vaccel_fpga_parallel_unpack, /* 18 */
        vaccel_fpga_vadd_unpack, /* 19 */
        vaccel_exec_with_res_unpack, /* 20 */
        vaccel_noop_unpack, /* 21 */
        vaccel_noop_unpack, /* 22 */
        vaccel_opencv_unpack, /* 23 */
        vaccel_foo_unpack, /* 24 */
};
```

Add the `vaccel_foo_unpack` function into this array and we can now save and close this file. And thats it for this directoy! Lets move on to the overall `src` directory once more.

Final thing to note here is to once more add the `#include "ops/foo.h"` in the `/src/vaccel.h.in` file and we are done!

Now lets make sure this operation works before we start working on the actual function itself.

---

## Testing the operation

We are just going to edit the noop plugin here as mentioned in the previous labs as most of the framework of the plugin is already written and it is much quicker to test it.
Would rather find out something is wrong instead of writing an entirely new plugin and trying to debug two things at once - not fun!

So, starting at the top of the `vaccel` directory, we need to go to `plugins/noop` directory and edit the `vaccel.c` file here:

```C
struct vaccel_op ops[] = {
        VACCEL_OP_INIT(ops[0], VACCEL_NO_OP, noop_noop),
        VACCEL_OP_INIT(ops[1], VACCEL_BLAS_SGEMM, noop_sgemm),
        VACCEL_OP_INIT(ops[2], VACCEL_IMG_CLASS, noop_img_class),
        VACCEL_OP_INIT(ops[3], VACCEL_IMG_DETEC, noop_img_detect),
        VACCEL_OP_INIT(ops[4], VACCEL_IMG_SEGME, noop_img_segme),
        VACCEL_OP_INIT(ops[5], VACCEL_IMG_POSE, noop_img_pose),
	VACCEL_OP_INIT(ops[6], VACCEL_IMG_DEPTH, noop_img_depth),
        VACCEL_OP_INIT(ops[7], VACCEL_EXEC, noop_exec),
        VACCEL_OP_INIT(ops[8], VACCEL_TF_SESSION_LOAD, noop_tf_session_load),
        VACCEL_OP_INIT(ops[9], VACCEL_TF_SESSION_RUN, noop_tf_session_run),
        VACCEL_OP_INIT(ops[10], VACCEL_TF_SESSION_DELETE,
                       noop_tf_session_delete),
        VACCEL_OP_INIT(ops[11], VACCEL_MINMAX, noop_minmax),
        VACCEL_OP_INIT(ops[12], VACCEL_F_ARRAYCOPY, v_arraycopy),
        VACCEL_OP_INIT(ops[13], VACCEL_F_VECTORADD, v_vectoradd),
        VACCEL_OP_INIT(ops[14], VACCEL_F_PARALLEL, v_parallel),
        VACCEL_OP_INIT(ops[15], VACCEL_F_MMULT, v_mmult),
        VACCEL_OP_INIT(ops[16], VACCEL_EXEC_WITH_RESOURCE,
                       noop_exec_with_resource),
        VACCEL_OP_INIT(ops[17], VACCEL_TORCH_JITLOAD_FORWARD,
                       noop_torch_jitload_forward),
        VACCEL_OP_INIT(ops[18], VACCEL_TORCH_SGEMM, noop_torch_sgemm),
        VACCEL_OP_INIT(ops[19], VACCEL_OPENCV, noop_opencv),
        VACCEL_OP_INIT(ops[20], VACCEL_TFLITE_SESSION_LOAD,
                       noop_tflite_session_load),
        VACCEL_OP_INIT(ops[21], VACCEL_TFLITE_SESSION_RUN,
                       noop_tflite_session_run),
        VACCEL_OP_INIT(ops[22], VACCEL_TFLITE_SESSION_DELETE,
                       noop_tflite_session_delete),
        VACCEL_OP_INIT(ops[23], VACCEL_FOO, noop_foo),
};

```
Here we are initialising the operations for the vaccel runtime and we can just add an extra element into the array `ops[]` as seen above as:

```C
VACCEL_OP_INIT(ops[23], VACCEL_FOO, noop_foo)
```

`VACCEL_FOO` is the operation we are creating and we have a function using this operation which we will call `noop_foo`.

Now we have to actually implement this `noop_foo` function otherwise we will not be able be use this operation.

```C
static int noop_foo(struct vaccel_session *sess, uint32_t a, uint32_t b,
                        uint32_t c)
{
        noop_debug("Calling foo for session %" PRId64 "", sess->id);
        return VACCEL_OK;
}

```
Paste this above the struct and it should be ready to go.
Great! Now, before testing our operation, we need to add the source and header files of the operation to `src/ops/meson.build` file in the following way:
```
vaccel_headers += files([
  'blas.h',
  'exec.h',
  'fpga.h',
  'genop.h',
  'image.h',
  'minmax.h',
  'noop.h',
  'opencv.h',
  'tf.h',
  'tflite.h',
  'torch.h',
  'foo.h',
])

vaccel_sources += files([
  'blas.c',
  'exec.c',
  'fpga.c',
  'genop.c',
  'image.c',
  'minmax.c',
  'noop.c',
  'opencv.c',
  'tf.c',
  'tflite.c',
  'torch.c',
  'foo.c',
])
```

Now we are ready to repeat the same exercise we did in lab1. Under the `vaccel` directory:

```bash
meson setup --prefix=<path> -Dplugin-noop=enabled build --reconfigure
meson compile -C build
meson install -C build
```
To run the executable from the `<path>` directory:
```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu/ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./lib/x86_64-linux-gnu/libvaccel-noop.so ./bin/noop
```

We can see in the debug console that the operation FOO should have been implementated:

```
2025.01.10-16:00:24.44 - <debug> Initializing vAccel
2025.01.10-16:00:24.44 - <info> vAccel 0.6.1-166-3cfa48b7-dirty
2025.01.10-16:00:24.44 - <debug> Created top-level rundir: /run/user/1009/vaccel/a8M11d
2025.01.10-16:00:24.44 - <info> Registered plugin noop 0.6.1-166-3cfa48b7-dirty
2025.01.10-16:00:24.44 - <debug> Registered function noop from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function sgemm from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function image classification from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function image detection from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function image segmentation from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function image pose estimation from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function image depth estimation from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function exec from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function TensorFlow session load from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function TensorFlow session run from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function TensorFlow session delete from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function MinMax from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Array copy from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Vector Add from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Parallel acceleration from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Matrix multiplication from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Exec with resource from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Torch jitload_forward function from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function Torch SGEMM from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function OpenCV Generic from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function TensorFlow Lite session load from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function TensorFlow Lite session run from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function TensorFlow Lite session delete from plugin noop
2025.01.10-16:00:24.44 - <debug> Registered function foo from plugin noop

```

## Function implementation

Now for the function implementation, we can just write out our own plugin which will then contain the function as needed:


As we did in lab2, create a new folder in the plugins directory and lets call it `foo`.

Once again, under the `plugins/foo` directory for `meson.build`:

```
foo_sources = files([
  'vaccel.c',
])

libvaccel_foo = shared_library('vaccel-foo',
  foo_sources,
  version: libvaccel_version,
  include_directories : include_directories('.'),
  c_args : plugins_c_args,
  dependencies : libvaccel_dep,
  install : true)
```

And for the function implementation `vaccel.c`:
Lets say for the purpose of this plugin it takes 2 variables, `a` and `b` and adds them together and then prints out the result - rather basic but you get the idea of it :) Here is the static API implementation of the function:

```C
#define _POSIX_C_SOURCE 200809L

#include "vaccel.h"
#include <inttypes.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define foo_debug(fmt, ...) vaccel_debug("[foo] " fmt, ##__VA_ARGS__)
#define foo_error(fmt, ...) vaccel_error("[foo] " fmt, ##__VA_ARGS__)

static int foo_foo(struct vaccel_session *sess, uint32_t a, uint32_t b,
                        uint32_t c)
{
        foo_debug("Calling foo for session %" PRId64 "", sess->id);
        foo_debug("Calling foo operation here");

        c = a + b;
        printf("Result = %d\n", c);

        foo_debug("Ending foo operation here");
        return VACCEL_OK;
}

struct vaccel_op ops[] = {
        VACCEL_OP_INIT(ops[0], VACCEL_FOO, foo_foo),
};

static int init(void)
{
        return register_plugin_functions(ops, sizeof(ops) / sizeof(ops[0]));
}

static int fini(void)
{
        return VACCEL_OK;
}

VACCEL_MODULE(.name = "foo", .version = VACCEL_VERSION,
              .vaccel_version = VACCEL_VERSION, .type = VACCEL_PLUGIN_DEBUG,
              .init = init, .fini = fini)

```

Now lets just create a basic `foo.c` in the examples folder, just like the hello world example:

```C
#include "vaccel.h"
#include <inttypes.h>
#include <stdint.h>
#include <stdio.h>

int main()
{
        int ret;
        struct vaccel_session sess;

        uint32_t a = 5, b = 5, c = 0;

        ret = vaccel_session_init(&sess, 0);
        if (ret != VACCEL_OK) {
                fprintf(stderr, "Could not initialize session\n");
                return 1;
        }

        printf("Initialized session with id: %" PRId64 "\n", sess.id);

        ret = vaccel_foo(&sess, a, b, c);
        if (ret) {
                fprintf(stderr, "Could not run op: %d\n", ret);
                goto close_session;
        }

close_session:
        if (vaccel_session_release(&sess) != VACCEL_OK) {
                fprintf(stderr, "Could not clear session\n");
                return 1;
        }

        return ret;
}
```

To build the plugin (as we did in lab2), add the following snippet to `plugins/meson.build`:
```
...
...
opt_plugin_foo = get_option('plugin-foo').enable_if(opt_tests.enabled()).enable_if(opt_plugins.enabled())
...
...
if opt_plugin_foo.enabled()
  subdir('foo')
endif

```
to `meson.build`:

```
...
summary({
...
  'Build the foo plugin': opt_plugin_foo.enabled(),
...
)

meson.add_dist_script(
...
  '-a', 'plugin-foo' + (opt_plugin_foo.enabled() ? 'enabled' : 'disabled'),
...
)

```

to `meson.options`:
```
option('plugin-foo',
  type : 'feature',
  value : 'auto',
  description : 'Build the foo debugging plugin')
```

Finally, we add the example source file `'foo.c'` to the end of the `examples_sources` table in the `examples/meson.build` file.

Now we can build our new plugin, by specifying the new option we just added:

```bash
meson setup -Dplugin-foo build --reconfigure
meson compile -C build
meson install -C build
```

And run it:

```bash
LD_LIBRARY_PATH=<path>/lib/x86_64-linux-gnu/ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=<path>/lib/x86_64-linux-gnu/libvaccel-foo.so <path>/bin/foo
```


And now for the result:

```
2025.01.10-17:02:37.48 - <debug> Initializing vAccel
2025.01.10-17:02:37.48 - <info> vAccel 0.6.1-166-3cfa48b7-dirty
2025.01.10-17:02:37.48 - <debug> Created top-level rundir: /run/user/1009/vaccel/pDGV3m
2025.01.10-17:02:37.48 - <info> Registered plugin foo 0.6.1-166-3cfa48b7-dirty
2025.01.10-17:02:37.48 - <debug> Registered function foo from plugin foo
2025.01.10-17:02:37.48 - <debug> Loaded plugin foo from /home/mgkeka/binaries/test/vaccel/lib/x86_64-linux-gnu/libvaccel-foo.so
2025.01.10-17:02:37.49 - <debug> New rundir for session 1: /run/user/1009/vaccel/pDGV3m/session.1
2025.01.10-17:02:37.49 - <debug> Initialized session 1
Initialized session with id: 1
2025.01.10-17:02:37.49 - <debug> Returning func from hint plugin foo
2025.01.10-17:02:37.49 - <debug> Found implementation in foo plugin
2025.01.10-17:02:37.49 - <debug> [foo] Calling foo for session 1
2025.01.10-17:02:37.49 - <debug> [foo] Calling foo operation here
Result = 10
2025.01.10-17:02:37.49 - <debug> [foo] Ending foo operation here
2025.01.10-17:02:37.49 - <debug> Released session 1
2025.01.10-17:02:37.49 - <debug> Shutting down vAccel
2025.01.10-17:02:37.49 - <debug> Cleaning up plugins
2025.01.10-17:02:37.49 - <debug> Unregistered plugin foo
```

Perfect, works as intended.

---

Hopefully this would give you a rather good idea of how to implement your own function into vAccel and for further examples just have a quick glance through some of the more complex examples in the corresponding directories.

