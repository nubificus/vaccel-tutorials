# Use vAccel to run your code

In this short tutorial, we will go through using the vAccel framework to execute 
an arbitrary piece of code. The process is straightforward: we build a simple 
library that wraps the call to our function into a vAccel `exec` operation and implement 
the respective wrapper to call the implementation of our function via the `exec` plugin.

The purpose of this tutorial is to showcase the simplicity of running a piece
of code via vAccel.

First lets go through the first step of implementing a simple wrapper library
for our code.

## Our code

Suppose we have some code written for a project and we want to integrate it in
vAccel, in order to expose the functionality to users without them caring about
the underlying implementation. 

In short, lets say we have a `vector add` operation, in `OpenCL`. A first-page
google result for `opencl example vector add` showed the following github repo:
`https://github.com/mantiuk/opencl_examples` which we forked at
`https://github.com/nubificus/opencl_examples`.

To build, you will need a working OpenCL installation. On a debian-based OS its
more than enough to do a simple: `apt-get install libpocl-dev`.

For this tutorial we will use a helper repo with the reference code. Lets clone
it and start:

```
git clone https://github.com/nubificus/vaccel-tutorial-code --recursive
cd vaccel-tutorial-code
```

The directory tree is the following:

```
$ tree -d .
.
├── app
│   └── opencl_examples
└── vaccelrt

```

In the `app` subfolder, we've added a git submodule pointing to the example
code mentioned earlier (`opencl_examples`). Lets go to the code and build it:

```
cd app/opencl_examples
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

The part we're interested in is the `vector add` executable, so lets run it!

```
$ ./vector_add/ocl_vector_add 
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
```

If we examine the
[code](https://github.com/nubificus/opencl_examples/blob/599c59e30d7fb74ff706b76fa6399e8e59f11112/vector_add/vector_add.cpp)
we'll see it is a simple [vector add implementation in
OpenCL](https://github.com/nubificus/opencl_examples/blob/599c59e30d7fb74ff706b76fa6399e8e59f11112/vector_add/vector_add.cpp#L47)
with the relevant [host
code](https://github.com/nubificus/opencl_examples/blob/599c59e30d7fb74ff706b76fa6399e8e59f11112/vector_add/vector_add.cpp#L84).

So, in order to use vAccel to execute this operation, we need to do two things:

- tweak the function and expose it as a shared library (`libify`) 
- wrap the function call with the vAccel `exec` call.

Lets see how easy it is to do both!

### "libify" the `vector add` operation

The simplest way to add an operation to vAccelRT is to "libify" this operation:
expose the operation as a function prototype through a shared library. To do
this, we need to tweak the build system of our example, and slightly change the
code.

The patch needed for `opencl_examples` is provided
[here](https://github.com/nubificus/vaccel-tutorial-code/blob/a6b10c7539d8b6158e7eea81760e95c798d10715/app/0001-Libify-vector_add.patch). 

In the opencl_examples directory simple run:

```
patch -p1 < ../0001-Libify-vector_add.patch
```

and rebuild:

```
cd build
make
```

This produces a `libvector_add.so` object which exposes a vector_add operation:

```
$ objdump -TC vector_add/libvector_add.so  | grep vector_add
vector_add/libvector_add.so:     file format elf64-x86-64
0000000000012a86 g    DF .text	0000000000000b83  Base        vector_add
```

To verify the library works as expected, we build a [small program](https://github.com/nubificus/vaccel-tutorial-code/blob/4b91749bbe756401c5d21f89a4ce4aaa50d43423/app/wrapper.c) that calls it. Let's navigate to the `app/` folder in our tutorial repo and build it:

```
cd app
make wrapper
```

Now, if we execute it we get the following:

```
$ ./wrapper 
./wrapper: error while loading shared libraries: libvector_add.so: cannot open shared object file: No such file or directory
```

This is because we haven't specified where the system can find the shared library containing the `vector_add` implementation. If we set `LD_LIBRARY_PATH` accordingly and re-run, we get the following:

```
$ LD_LIBRARY_PATH=opencl_examples/build/vector_add/ ./wrapper 
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
```

Aha! We have a library, `libvector_add.so`, a wrapper executable
that calls this library, and we're a bit familiar with vAccel so lets glue
these together! 

### vAccelRT integration

To run abritrary function calls using vAccel, we need to use vAccel's operation
`exec`. In short, `exec`, is a way to execute arbitrary operations, given an
internal (user-defined, framework agnostic) representation of functions and
arguments, provided the implementation is exposed as a symbol through a shared
library.

We build a frontend library that wraps all vAccel operations under a
`vector_add()` function call. The
[code](https://github.com/nubificus/vaccel-tutorial-code/blob/4b91749bbe756401c5d21f89a4ce4aaa50d43423/app/wrapper_exec.c)
is provided in the repo. 

Essentially, what the code does, is the following:

- [initialize a `vaccel session`](https://github.com/nubificus/vaccel-tutorial-code/blob/4b91749bbe756401c5d21f89a4ce4aaa50d43423/app/wrapper_exec.c#L14)
- [specify the shared object and the symbol name](https://github.com/nubificus/vaccel-tutorial-code/blob/4b91749bbe756401c5d21f89a4ce4aaa50d43423/app/wrapper_exec.c#L22)
- [call `vaccel_exec` with these arguments](https://github.com/nubificus/vaccel-tutorial-code/blob/4b91749bbe756401c5d21f89a4ce4aaa50d43423/app/wrapper_exec.c#L25)
- [finalize the session](https://github.com/nubificus/vaccel-tutorial-code/blob/4b91749bbe756401c5d21f89a4ce4aaa50d43423/app/wrapper_exec.c#L32)


We build the library:

```
make libwrapper.so
```

and the same wrapper program, only this time, we link against `libwrapper.so`
instead of `libvector_add.so`:

```
make wrapper-vaccel
```

We now point the `LD_LIBRARY_PATH` variable to the path `libwrapper.so` exists and we run the program:

```
LD_LIBRARY_PATH=. ./wrapper-vaccel 
./wrapper-vaccel: error while loading shared libraries: libvaccel.so: cannot open shared object file: No such file or directory
```

This means that we need to tell the loader where to find libvaccel.so.
Additionally, we need to specify a vAccel backend via the `VACCEL_BACKENDS`
enviromental variable. For more info have a look at the
[lab1](https://github.com/nubificus/vaccel-tutorials/tree/main/lab1) and
[lab2](https://github.com/nubificus/vaccel-tutorials/tree/main/lab2) tutorials
in this repo.

In short:
```
cd ../vaccelrt
mkdir -p build
cd build
cmake ../ -DBUILD_PLUGIN_EXEC=ON
make
cd ../app
```

After we've built vAccel and the vAccel exec plugin, we can run our wrapper:

```
$ LD_LIBRARY_PATH=.:../vaccelrt/build/src/ VACCEL_BACKENDS=../vaccelrt/build/plugins/exec/libvaccel-exec.so ./wrapper-vaccel 
Initialized session with id: 1
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
```

We see that the execution output is exactly the same, only there's a debug
message regarding the session initialization we've added in our frontend
wrapper library.

If we enable debug, we can see what's going on under the hood:

```
VACCEL_DEBUG_LEVEL=4 LD_LIBRARY_PATH=.:../vaccelrt/build/src/ VACCEL_BACKENDS=../vaccelrt/build/plugins/exec/libvaccel-exec.so ./wrapper-vaccel 
2021.04.09-12:16:15.11 - <debug> Initializing vAccel
2021.04.09-12:16:15.11 - <debug> Registered plugin exec
2021.04.09-12:16:15.11 - <debug> Registered function noop from plugin exec
2021.04.09-12:16:15.11 - <debug> Registered function exec from plugin exec
2021.04.09-12:16:15.11 - <debug> Loaded plugin exec from ../vaccelrt/build/plugins/exec/libvaccel-exec.so
2021.04.09-12:16:15.11 - <debug> session:1 New session
Initialized session with id: 1
2021.04.09-12:16:15.11 - <debug> session:1 Looking for plugin implementing exec
2021.04.09-12:16:15.11 - <debug> Found implementation in exec plugin
2021.04.09-12:16:15.11 - <debug> Calling exec for session 1
2021.04.09-12:16:15.11 - <debug> [exec] library: opencl_examples/build/vector_add/libvector_add.so
2021.04.09-12:16:15.12 - <debug> [exec] symbol: vector_add
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
2021.04.09-12:16:15.44 - <debug> session:1 Free session
2021.04.09-12:16:15.44 - <debug> Shutting down vAccel
2021.04.09-12:16:15.44 - <debug> Cleaning up plugins
2021.04.09-12:16:15.44 - <debug> Unregistered plugin exec
```

First vAccel is initialized. Then the libvaccel-exec.so plugin is registered,
implementing two operations: `noop` and `exec`. Then a request for a new
session is received, initialized and the call to `vaccel_exec` from the wrapper
library gets triggered. The system looks for an implementation of vaccel_exec
and forwards the call to the relevant plugin (the `exec` in this case. The
plugin implementation parses the library argument and opens it; then, it gets
the symbol address from this library and executes it. Finally, it cleans up the
session and exits.

### Adding arguments

Well, calling a simple function with no arguments seemed a bit too naive right?
How about we tweak the `vector_add` example so that we pass the vectors to be
calculated as arguments?

#### Tweak `vector_add`

First we need to change our [wrapper
program](https://github.com/nubificus/vaccel-tutorial-code/blob/a6b10c7539d8b6158e7eea81760e95c798d10715/app/wrapper-args.c)
to call vector_add with arguments.

We hardcode the input, two Vectors, A and B, and allocate an empty vector for the result. The last parameter is the dimension of the vectors. Our function prototype now looks like the following:

```
int vector_add(int *A, int *B, int *C, int dimension);
```

In order to be agnostic about the parameters of the function, we pack them using a wrapper library and unpack them in our implementation code.

The
[patch](https://github.com/nubificus/vaccel-tutorial-code/blob/a6b10c7539d8b6158e7eea81760e95c798d10715/app/0002-Add-arguments-generalize-vector_add.patch) for the `vector_add` library
is provided in the repo. To apply, run the following:

```
cd app/opencl_examples/
patch -p1 < ../0002-Add-arguments-generalize-vector_add.patch
```

and rebuild:

```
cd build
make
```

Finally, we need to tweak the frontend library to translate `vector_add` to `vaccel_exec`. The [new library](https://github.com/nubificus/vaccel-tutorial-code/blob/a6b10c7539d8b6158e7eea81760e95c798d10715/app/wrapper_exec-args.c) is in the repo. Let's build it, along with the wrapper program!

```
cd app
make libwrapper-args.so
make wrapper-args-vaccel
```

Notice the [change of
placement](https://github.com/nubificus/vaccel-tutorial-code/blob/a6b10c7539d8b6158e7eea81760e95c798d10715/app/wrapper_exec-args.c#L23)
for the libvector_add.so. Let's copy the binary there:

```
cp opencl_examples/build/vector_add/libvector_add.so /tmp
```

and run our program:

```
$ LD_LIBRARY_PATH=.:../vaccelrt/build/src/ VACCEL_BACKENDS=../vaccelrt/build/plugins/exec/libvaccel-exec.so ./wrapper-args-vaccel 
Initialized session with id: 1
Using platform: Intel(R) OpenCL HD Graphics
Using device: Intel(R) Graphics [0x9b41]
 result: 
0 2 4 3 5 7 6 8 10 9 
Operation successful!
A:  0  1  2  3  4  5  6  7  8  9 
+
B:  0  1  2  0  1  2  0  1  2  0 
=
C:  0  2  4  3  5  7  6  8 10  9 
```

Amazing right? :P

We are able to abstract away the execution of a simple operation via vAccel!
