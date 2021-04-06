# Add an operation

In this short tutorial, we will go through adding a simple vector add operation
to vAccelRT, and build a plugin that implements this operation using OpenCL.
The purpose of this tutorial is to showcase the simplicity of adding an
operation and how straightforward adding a new plugin to support an operation
is.


## Our code

Suppose we have some code written for a project and we want to integrate it in
vAccel, in order to expose the functionality to users without them caring about
the underlying implementation. 

In short, lets say we have a `vector add` operation, in `OpenCL`. A first-page
google result for `opencl example vector add` showed the following github repo:
`https://github.com/mantiuk/opencl_examples` which we forked at
`https://github.com/nubificus/opencl_examples`.

To build, you will need a working OpenCL installation. On my debian-based OS I
just did `apt-get install libpocl-dev`.

So, lets get the code and build it:

```
git clone https://github.com/nubificus/opencl_examples
cd opencl_examples
mkdir build
cd build
cmake ../
make
```

There should be two executables lying around in each of the example folders:

```
$ find . -path ./CMakeFiles -prune -o -type f -executable
./list_platforms/ocl_list_platforms
./vector_add/ocl_vector_add

```

let's execute ocl_vector_add:

```
$ ./vector_add/ocl_vector_add 
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
```

If we examine the code we'll see it is a simple vector add implementation in
OpenCL with the relevant host code:

```
#include <CL/cl.hpp>
#include <iostream>

int main(){
[...]
		std::string kernel_code =
			"__kernel void simple_add(__global const int* A, __global const int* B, __global int* C) {"
			"	int index = get_global_id(0);"
			"	C[index] = A[index] + B[index];"
			"};";
[...]
		int A[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
		int B[] = { 0, 1, 2, 0, 1, 2, 0, 1, 2, 0 };

		//create queue to which we will push commands for 	the device.
		cl::CommandQueue queue(context, default_device);

		//write arrays A and B to the device
		queue.enqueueWriteBuffer(buffer_A, CL_TRUE, 0, sizeof(int) * 10, A);
		queue.enqueueWriteBuffer(buffer_B, CL_TRUE, 0, sizeof(int) * 10, B);

		cl::Kernel kernel(program, "simple_add");

		kernel.setArg(0, buffer_A);
		kernel.setArg(1, buffer_B);
		kernel.setArg(2, buffer_C);
		queue.enqueueNDRangeKernel(kernel, cl::NullRange, cl::NDRange(10), cl::NullRange);


		int C[10];
		//read result C from the device to array C
		queue.enqueueReadBuffer(buffer_C, CL_TRUE, 0, sizeof(int) * 10, C);
		queue.finish();

		std::cout << " result: \n";
		for (int i = 0; i < 10; i++){
			std::cout << C[i] << " ";
		}
		std::cout << std::endl;
	}
	catch (cl::Error err)
	{
		printf("Error: %s (%d)\n", err.what(), err.err());
		return -1;
	}

    return 0;
}	
```

The math is correct, so we're fine :D

So, if we want to expose this `vector add` operation via vAccelRT, we need to
do two things:

- expose the function prototype as a vAccel operation, in this case `int
  vector_add()`
- transform this code into a vAccel plugin.

Lets see how easy it is to do both!

### "libify" the `vector add` operation

The simplest way to add an operation to vAccelRT is to "libify" this operation:
expose the operation as a function prototype through a shared library. To do
this, we need to tweak the build system, and change the code slightly.

The patch to the repo is the following:

```
diff --git a/vector_add/CMakeLists.txt b/vector_add/CMakeLists.txt
index 714509e..dab44e6 100755
--- a/vector_add/CMakeLists.txt
+++ b/vector_add/CMakeLists.txt
@@ -1,6 +1,7 @@
 include_directories ("${PROJECT_SOURCE_DIR}/include" ${OpenCL_INCLUDE_DIRS})
 
-add_executable(ocl_vector_add vector_add.cpp)
-target_compile_features(ocl_vector_add PRIVATE cxx_range_for)
-target_link_libraries(ocl_vector_add ${OpenCL_LIBRARIES})
+add_library(vector_add SHARED vector_add.cpp)
+target_compile_options(vector_add PUBLIC -Wall -Wextra )
+set_property(TARGET vector_add PROPERTY LINK_FLAGS "-lOpenCL -shared")
+
 
diff --git a/vector_add/vector_add.cpp b/vector_add/vector_add.cpp
index 5451547..f5ab3e0 100755
--- a/vector_add/vector_add.cpp
+++ b/vector_add/vector_add.cpp
@@ -17,7 +17,8 @@
 #include <CL/cl.hpp>
 #include <iostream>
 
-int main(){
+//int main(){
+extern "C" int vector_add(){
        try{
 
                //get all platforms (drivers)
```

a re-build of our code produces a shared object, `libvector_add.so`, which
contains a symbol `vector_add`, which is, essentially, the vector add
operation.

To verify the library works correctly, we build a small program that calls it:

```
int vector_add();

int main(int argc, char **argv)
{

	vector_add();

	return 0;
}
```

we build it with the following arguments:

```
gcc wrapper.c -Wall -L../opencl_examples/build/vector_add/ -lvector_add -lOpenCL -o vector_add
```

and run it, making sure we specify the path to libvector_add.so:

```
# LD_LIBRARY_PATH=$(PWD)/../opencl_examples/build/vector_add/ ./vector_add
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
```

All fine until now: we have a library, `libvector_add.so`, a wrapper executable
that calls this library, and we're a bit familiar with vAccel so lets glue
these together! 

### vAccelRT integration

To add a new vAccel operation we need to:

- add it as an operation in `vaccel_ops.{c,h}`
- add its plugin code to `plugins/<operation>/vaccel.c`

Luckily, vAccelRT has support for a `generic operation`. This means that given
an internal (user-defined, framework agnostic) representation of functions and
arguments, the user can execute arbitrary functions, provided they are exposed
through a shared library.

Let's tweak our wrapper program, to use `vaccel_genop` (`wrapper_genop.c`):

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <vaccel.h>
#include <vaccel_ops.h>

int vaccel_vector_add()
{

	int ret = 0;
	struct vaccel_session sess;

        ret = vaccel_sess_init(&sess, 0);
        if (ret != VACCEL_OK) {
                fprintf(stderr, "Could not initialize session\n");
                return 1;
        }

        printf("Initialized session with id: %u\n", sess.session_id);

	char *operation = "vector-add";
	size_t len = strlen(operation);

        ret = vaccel_genop(&sess, NULL, operation, 0, len);
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

int main(int argc, char **argv)
{
	vaccel_vector_add();

	return 0;
}
```

We've added the vaccel_vector_add() function which essentially initializes a
vaccel session, calls `vaccel_genop` with an input parameter string
`vector-add` and closes the session to return.

We build it, linking against vaccelrt:

```
gcc -owrapper_genop wrapper_genop.c -Wall -L../vaccelrt/build/src -I../vaccelrt/src -lvaccel -ldl
```

If we try to execute it with the `noop` plugin and some debug enabled, we get:

```
$ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=../vaccelrt/build/plugins/noop/libvaccel-noop.so LD_LIBRARY_PATH=../vaccelrt/build/src ./a.out 
2021.04.03-07:06:21.20 - <debug> Initializing vAccel
2021.04.03-07:06:21.20 - <debug> Registered plugin noop
[...]
2021.04.03-07:06:21.20 - <debug> Registered function gen-op from plugin noop
2021.04.03-07:06:21.20 - <debug> Loaded plugin noop from ../vaccelrt/build/plugins/noop/libvaccel-noop.so
2021.04.03-07:06:21.20 - <debug> session:1 New session
Initialized session with id: 1
2021.04.03-07:06:21.20 - <debug> session:1 Looking for plugin implementing generic op
2021.04.03-07:06:21.20 - <debug> Found implementation in noop plugin
2021.04.03-07:06:21.20 - <debug> Calling do-op for session 1
2021.04.03-07:06:21.20 - <debug> [noop] [genop] in_nargs: 10, out_nargs: 0

2021.04.03-07:06:21.20 - <debug> session:1 Free session
2021.04.03-07:06:21.20 - <debug> Shutting down vAccel
2021.04.03-07:06:21.20 - <debug> Cleaning up plugins
2021.04.03-07:06:21.20 - <debug> Unregistered plugin noop
```

We see that the `genop` operation has been triggered and got 10 bytes as an
input argument.

In order to take advantage of this generic operation, we should write our own
plugin, that just calls vector_add via the shared library. Let's see how we can
do that!

### Transform our code to a vAccel plugin

To integrate our code to vAccel as a plugin we need to:

- build it as a shared library and replace `main` with the name of our
  choosing; in this case `vector_add`,
- create the glue code to parse the `vaccel_genop` input arguments.

Let's use the `noop` plugin as reference; we copy its contents to a new
directory, called `vadd`. 

We go back to the vAccelRT base source dir and execute the following:

```
cp -avf plugins/noop plugins/vadd
```

we get rid of any noop mentions in there so we come up with the following directory structure:

```
plugins/vadd
├── CMakeLists.txt
└── vaccel.c
```

copy `libvector_add.so` to this directory too.

and the following contents for CMakeLists.txt:

```
set(include_dirs ${CMAKE_SOURCE_DIR}/src)
set(SOURCES vaccel.c ${include_dirs}/vaccel.h ${include_dirs}/plugin.h)

add_library(vaccel-vadd SHARED ${SOURCES})
target_include_directories(vaccel-vadd PRIVATE ${include_dirs})

target_link_libraries(vaccel-vadd PRIVATE vector_add OpenCL)

# Setup make install
install(TARGETS vaccel-vadd DESTINATION "${lib_path}")
```

and vaccel.c

```
#include <stdio.h>
#include <plugin.h>

#include "log.h"
int vector_add();

int ocl_vector_add(struct vaccel_session *sess)
{
	vaccel_debug("[vadd] session:%u ",
			sess->session_id);

	return vector_add();
}

struct vaccel_op op = VACCEL_OP_INIT(op, VACCEL_GEN_OP, ocl_vector_add);

static int init(void)
{
	return register_plugin_function(&op);
}

static int fini(void)
{
	return VACCEL_OK;
}

VACCEL_MODULE(
	.name = "vadd",
	.version = "0.1",
	.init = init,
	.fini = fini
)
```

We build vAccelRT and we see that a new plugin is now available,
libvaccel-vadd.so. Lets use this one instead of the `noop` one!

```
VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=../vaccelrt/build/plugins/vadd/libvaccel-vadd.so LD_LIBRARY_PATH=../vaccelrt/build/src:. ./wrapper_genop
2021.04.03-07:16:55.77 - <debug> Initializing vAccel
2021.04.03-07:16:55.78 - <debug> Registered plugin vadd
2021.04.03-07:16:55.78 - <debug> Registered function gen-op from plugin vadd
2021.04.03-07:16:55.78 - <debug> Loaded plugin vadd from ../vaccelrt/build2/plugins/vadd/libvaccel-vadd.so
2021.04.03-07:16:55.78 - <debug> session:1 New session
Initialized session with id: 1
2021.04.03-07:16:55.78 - <debug> session:1 Looking for plugin implementing generic op
2021.04.03-07:16:55.78 - <debug> Found implementation in vadd plugin
2021.04.03-07:16:55.78 - <debug> Calling do-op for session 1
2021.04.03-07:16:55.78 - <debug> [vadd] [genop] in_nargs: 10, out_nargs: 0

Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
2021.04.03-07:16:56.10 - <debug> session:1 Free session
2021.04.03-07:16:56.10 - <debug> Shutting down vAccel
2021.04.03-07:16:56.10 - <debug> Cleaning up plugins
2021.04.03-07:16:56.10 - <debug> Unregistered plugin vadd
```

We made it! We are actually able to call vector_add() from vAccel by following
three simple steps:

- "libifying" our code
- creating a simple plugin based on this library
- modifying our calling code to initialize a vaccel session and call
  `vaccel_genop`

### Adding arguments

Well, calling a simple function with no arguments seemed a bit too naive right?
How about we tweak the `vector_add` example so that we pass the vectors to be
calculated as arguments?

#### Tweak `vector_add`

First we need to change our library code:

```
diff --git a/vector_add/CMakeLists.txt b/vector_add/CMakeLists.txt
index 714509e..dab44e6 100755
--- a/vector_add/CMakeLists.txt
+++ b/vector_add/CMakeLists.txt
@@ -1,6 +1,7 @@
 include_directories ("${PROJECT_SOURCE_DIR}/include" ${OpenCL_INCLUDE_DIRS})
 
-add_executable(ocl_vector_add vector_add.cpp)
-target_compile_features(ocl_vector_add PRIVATE cxx_range_for)
-target_link_libraries(ocl_vector_add ${OpenCL_LIBRARIES})
+add_library(vector_add SHARED vector_add.cpp)
+target_compile_options(vector_add PUBLIC -Wall -Wextra )
+set_property(TARGET vector_add PROPERTY LINK_FLAGS "-lOpenCL -shared")
+
 
diff --git a/vector_add/vector_add.cpp b/vector_add/vector_add.cpp
index 5451547..d1a29a1 100755
--- a/vector_add/vector_add.cpp
+++ b/vector_add/vector_add.cpp
@@ -16,10 +16,15 @@
 
 #include <CL/cl.hpp>
 #include <iostream>
+//int A[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
+//int B[] = { 0, 1, 2, 0, 1, 2, 0, 1, 2, 0 };
 
-int main(){
+extern "C" int vector_add (int* A, int* B, int* C, int dimension){
 	try{
 
+		for (int i = 0; i < dimension; i++){
+			std::cout << A[i] << " ";
+		}
 		//get all platforms (drivers)
 		std::vector<cl::Platform> all_platforms;
 		cl::Platform::get(&all_platforms);
@@ -27,7 +32,7 @@ int main(){
 			std::cout << " No platforms found. Check OpenCL installation!\n";
 			exit(1);
 		}
-		cl::Platform default_platform = all_platforms[0];
+		cl::Platform default_platform = all_platforms[atoi(getenv("OCL_DEVICE_NR"))];
 		std::cout << "Using platform: " << default_platform.getInfo<CL_PLATFORM_NAME>() << "\n";
 
 		//get default device of the default platform
@@ -62,34 +67,32 @@ int main(){
 		}
 
 		// create buffers on the device
-		cl::Buffer buffer_A(context, CL_MEM_READ_WRITE, sizeof(int) * 10);
-		cl::Buffer buffer_B(context, CL_MEM_READ_WRITE, sizeof(int) * 10);
-		cl::Buffer buffer_C(context, CL_MEM_READ_WRITE, sizeof(int) * 10);
+		cl::Buffer buffer_A(context, CL_MEM_READ_WRITE, sizeof(int) * dimension);
+		cl::Buffer buffer_B(context, CL_MEM_READ_WRITE, sizeof(int) * dimension);
+		cl::Buffer buffer_C(context, CL_MEM_READ_WRITE, sizeof(int) * dimension);
 
-		int A[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
-		int B[] = { 0, 1, 2, 0, 1, 2, 0, 1, 2, 0 };
 
 		//create queue to which we will push commands for 	the device.
 		cl::CommandQueue queue(context, default_device);
 
 		//write arrays A and B to the device
-		queue.enqueueWriteBuffer(buffer_A, CL_TRUE, 0, sizeof(int) * 10, A);
-		queue.enqueueWriteBuffer(buffer_B, CL_TRUE, 0, sizeof(int) * 10, B);
+		queue.enqueueWriteBuffer(buffer_A, CL_TRUE, 0, sizeof(int) * dimension, A);
+		queue.enqueueWriteBuffer(buffer_B, CL_TRUE, 0, sizeof(int) * dimension, B);
 
 		cl::Kernel kernel(program, "simple_add");
 
 		kernel.setArg(0, buffer_A);
 		kernel.setArg(1, buffer_B);
 		kernel.setArg(2, buffer_C);
-		queue.enqueueNDRangeKernel(kernel, cl::NullRange, cl::NDRange(10), cl::NullRange);
+		queue.enqueueNDRangeKernel(kernel, cl::NullRange, cl::NDRange(dimension), cl::NullRange);
 
-		int C[10];
+		//int C[dimension];
 		//read result C from the device to array C
-		queue.enqueueReadBuffer(buffer_C, CL_TRUE, 0, sizeof(int) * 10, C);
+		queue.enqueueReadBuffer(buffer_C, CL_TRUE, 0, sizeof(int) * dimension, C);
 		queue.finish();
 
 		std::cout << " result: \n";
-		for (int i = 0; i < 10; i++){
+		for (int i = 0; i < dimension; i++){
 			std::cout << C[i] << " ";
 		}
 		std::cout << std::endl;
```

This patch, allows for 4 arguments, 3 for the input/output vectors and 1 for
the dimension.

Let's rebuild and test out our new `vector_add` library with a tweaked wrapper program:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int vector_add(int *A, int *B, int *C, int dimension);

int main(int argc, char **argv)
{
	int A[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
	int B[] = { 0, 1, 2, 0, 1, 2, 0, 1, 2, 0 };
	int C[] = { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 };
	int dimension = sizeof(A)/sizeof(int);
	int i = 0;

	vector_add(A, B, C, dimension);

	printf("wrapper start printing vector C\n");
	for (i=0;i<dimension;i++)
		printf("%d ", C[i]);
	printf("\n");

	return 0;
}
```

```
gcc wrapper-args.c -Wall -L../opencl_examples/build/vector_add -lvector_add -lOpenCL
```

```
$ OCL_DEVICE_NR=0 LD_LIBRARY_PATH=../opencl_examples/build/vector_add/ ./a.out 
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
wrapper start printing vector C
0 2 4 3 5 7 6 8 10 9 
```

All good for now. Let's tweak vAccel.

#### Pack/unpack arguments

First thing to do, is come up with a representation for arguments in order to
easily pack them in the wrapper program and unpack them in the plugin. Something 
along the lines of the following is more than enough:

```
struct vector_arg {
        size_t len;
        uint8_t *buf;
};
```

Let's tweak our vAccel wrapper program:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#include <vaccel.h>
#include <vaccel_ops.h>

struct vector_arg {
        size_t len;
        uint8_t *buf;
} __attribute__ ((packed));

int vaccel_vector_add(int *A, int *B, int *C, int dimension)
{

	int ret = 0;
	struct vector_arg op_args[5];
	struct vaccel_session sess;

        ret = vaccel_sess_init(&sess, 0);
        if (ret != VACCEL_OK) {
                fprintf(stderr, "Could not initialize session\n");
                return 1;
        }

        printf("Initialized session with id: %u\n", sess.session_id);

	char *operation = "vector-add";
	size_t oplen = strlen(operation);

        memset(op_args, 0, sizeof(op_args));

        op_args[0].len = oplen;
        op_args[0].buf = operation;
        op_args[1].len = dimension * sizeof(int);
        op_args[1].buf = A;
        op_args[2].len = dimension * sizeof(int);
        op_args[2].buf = B;
        op_args[3].len = sizeof(int);
        op_args[3].buf = &dimension;

        op_args[4].len = dimension * sizeof(int);
        op_args[4].buf = C;

        ret = vaccel_genop(&sess, &op_args[4], &op_args[0], 1, 4);
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

int main(int argc, char **argv)
{
	int A[] = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };
	int B[] = { 0, 1, 2, 0, 1, 2, 0, 1, 2, 0 };
	int C[] = { 0, 0, 0, 0, 0, 0, 0, 0, 0, 0 };
	int dimension = sizeof(A)/sizeof(int);
	int i = 0;

	vaccel_vector_add(A, B, C, dimension);

	printf("wrapper start printing vector C\n");
	for (i=0;i<dimension;i++)
		printf("%d ", C[i]);
	printf("\n");

	return 0;
}
```

Before running it, we need to make sure we also tweak the plugin we built, in
order to properly unpack the arguments.

Lets go to the vaccelrt source tree and edit `plugins/vadd/vaccel.c` into:
```
#include <stdio.h>
#include <plugin.h>
#include <byteswap.h>

#include "log.h"

int vector_add(int*A,int*B,int*C, int dimension);

struct vector_arg {
        size_t len;
        uint8_t *buf;
} __attribute__ ((packed));

static int noop(struct vaccel_session *session)
{
	vaccel_debug("Calling no-op for session %u", session->session_id);

	vaccel_debug("[vadd] [noop] \n");

	return VACCEL_OK;
}

static int genop(struct vaccel_session *session, void *out_args, void *in_args,
			  size_t out_nargs, size_t in_nargs)
{
	int i = 0;
	struct vector_arg *in_arg = in_args;
	struct vector_arg *out_arg = out_args;

	vaccel_debug("Calling do-op for session %u", session->session_id);

	vaccel_debug("[vadd] [genop] in_nargs: %d, out_nargs: %d\n", in_nargs, out_nargs);

	int dimension = *(in_arg[3].buf);
	int *A = (int*)in_arg[1].buf;
	int *B = (int*)in_arg[2].buf;
	int *C = (int*)out_arg[0].buf;
	vector_add(A, B, C, dimension);

	return VACCEL_OK;
}

struct vaccel_op ops[] = {
	VACCEL_OP_INIT(ops[0],VACCEL_NO_OP, noop),
	VACCEL_OP_INIT(ops[1],VACCEL_GEN_OP, genop),
};

static int init(void)
{
	return register_plugin_functions(ops, sizeof(ops) / sizeof(ops[0]));
}

static int fini(void)
{
	return VACCEL_OK;
}

VACCEL_MODULE(
	.name = "vadd",
	.version = "0.1",
	.init = init,
	.fini = fini
)
```

and make sure we build again in `vaccelrt/build`:

```
make
```

This should produce an updated `libvaccel-vadd.so`.

Now, lets go back to our wrapper program and build it:
```
gcc -owrapper_genop-args wrapper_genop-args.c -Wall -L../vaccelrt/build/src -I../vaccelrt/src -lvaccel -ldl
```

If we run it, we get the following:
```
VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=../vaccelrt/build2/plugins/vadd/libvaccel-vadd.so LD_LIBRARY_PATH=../vaccelrt/build2/src:. ./wrapper_genop 
2021.04.06-20:34:32.22 - <debug> Initializing vAccel
2021.04.06-20:34:32.23 - <debug> Registered plugin vadd
2021.04.06-20:34:32.23 - <debug> Registered function noop from plugin vadd
2021.04.06-20:34:32.23 - <debug> Registered function gen-op from plugin vadd
2021.04.06-20:34:32.23 - <debug> Loaded plugin vadd from ../vaccelrt/build2/plugins/vadd/libvaccel-vadd.so
2021.04.06-20:34:32.23 - <debug> session:1 New session
Initialized session with id: 1
2021.04.06-20:34:32.23 - <debug> session:1 Looking for plugin implementing generic op
2021.04.06-20:34:32.23 - <debug> Found implementation in vadd plugin
2021.04.06-20:34:32.23 - <debug> Calling do-op for session 1
2021.04.06-20:34:32.23 - <debug> [vadd] [genop] in_nargs: 4, out_nargs: 1

Using platform: Portable Computing Language
Using device: pthread-Intel(R) Core(TM) i7-10610U CPU @ 1.80GHz
 result: 
0 2 4 3 5 7 6 8 10 9 
2021.04.06-20:34:32.33 - <debug> session:1 Free session
wrapper start printing vector C
0 2 4 3 5 7 6 8 10 9 
2021.04.06-20:34:32.33 - <debug> Shutting down vAccel
2021.04.06-20:34:32.33 - <debug> Cleaning up plugins
2021.04.06-20:34:32.33 - <debug> Unregistered plugin vadd
```

Which seems like a successful execution. We did it!
