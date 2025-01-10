# lab2: How to write a vAccel plugin

In [lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md)
we saw how to build and execute a simple vAccel application calling the
`vaccel_noop` function which is implemented by the `noop` plugin.

In this lab, we will go through the process of writing our own vAccel plugin
that implements the `vaccel_noop` that we used in our "Hello, world" example in
[lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md).
The purpose of this exercise is to showcase the rationale of the plugin/backend
system and the simplicity of adding a new plugin to vAccel.

To implement a vAccel plugin, all we need to do is add the implementation of
the relevant frontend function and register it to the vAccel plugin subsystem.

To start, we assume we have built and familiarized ourselves with vAccel using
[lab1](https://github.com/nubificus/vaccel-tutorials/blob/main/lab1/README.md).


We will use the `noop` plugin as a skeleton for our new plugin which we will
call `helloworld`. For simplicity, starting from the top-level of the `vaccel`
repo, we copy the `noop` plugin directory structure to a new directory:

```
cp -avf plugins/noop plugins/helloworld
```

We replace `noop` with the name of our choosing (`helloworld`) so we come up with
the following directory structure:

```
tree -L 1 plugins/helloworld
plugins/helloworld
├── meson.build
└── vaccel.c
```

We need to change the contents of `plugins/helloworld/meson.build` to reflect
the `noop` to `helloworld` change:

```meson
helloworld_sources = files([
  'vaccel.c',
])

libvaccel_helloworld = shared_library('vaccel-helloworld',
  helloworld_sources,
  version: libvaccel_version,
  include_directories : include_directories('.'),
  c_args : plugins_cargs,
  dependencies : libvaccel_private,
  install : true)
```

Similarly, we replace the plugin implementation of `noop_noop` function with the `helloworld` function in `vaccel.c`,
adding a simple message:

```C
#define helloworld_debug(fmt, ...) vaccel_debug("[helloworld] " fmt, ##__VA_ARGS__)
#define helloworld_error(fmt, ...) vaccel_error("[helloworld] " fmt, ##__VA_ARGS__)

static int helloworld_helloworld(struct vaccel_session *sess)
{
        helloworld_debug("Calling vaccel-helloworld for session %" PRId64 "", sess->id);

        return VACCEL_OK;
}

...

struct vaccel_op ops[] = {
	VACCEL_OP_INIT(ops[0], VACCEL_NO_OP, helloworld_helloworld),
	...
}

```
Also, run the following command to replace all the occurrences of `noop`.
```
sed -i 's/noop/helloworld/g' plugins/helloworld/vaccel.c
```

Additionally, we need to add the relevant rules in the `meson.build` files
to build the plugin along with the rest of the vAccel code.

We add the next lines to `plugins/meson.build`:
```
...
...
opt_plugin_helloworld = get_option('plugin-helloworld').enable_if(opt_tests.enabled()).enable_if(opt_plugins.enabled())
...
...
if opt_plugin_helloworld.enabled()
  subdir('helloworld')
endif

```

To `meson.build`:
```
...

summary({
...
'Build the helloworld plugin': opt_plugin_helloworld.enabled(),
...
)

meson.add_dist_script(
...
 '-a', 'plugin-helloworld' + (opt_plugin_helloworld.enabled() ? 'enabled' : 'disabled'),
...
)

```

And to `meson.options`:
```
option('plugin-helloworld',
  type : 'feature',
  value : 'auto',
  description : 'Build the helloworld debugging plugin')
```

Now we can build our new plugin, by specifying the new option we just added:

```
meson setup build -Dplugin-helloworld --reconfigure
meson compile -C build
meson install -C build
```

Under the `<path>` directory we see that a new plugin is now available, `libvaccel-helloworld.so`. Lets use this one instead of the `noop` one!

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu/ VACCEL_DEBUG_LEVEL=4 VACCEL_BACKENDS=./lib/x86_64-linux-gnu/libvaccel-helloworld.so ./bin/noop
```
should return:
```
2025.01.09-16:07:18.11 - <debug> Initializing vAccel
2025.01.09-16:07:18.11 - <info> vAccel 0.6.1-166-3cfa48b7-dirty
2025.01.09-16:07:18.11 - <debug> Created top-level rundir: /run/user/1009/vaccel/jRcbta
2025.01.09-16:07:18.11 - <info> Registered plugin helloworld 0.6.1-166-3cfa48b7-dirty
2025.01.09-16:07:18.11 - <debug> Registered function noop from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function sgemm from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function image classification from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function image detection from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function image segmentation from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function image pose estimation from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function image depth estimation from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function exec from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function TensorFlow session load from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function TensorFlow session run from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function TensorFlow session delete from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function MinMax from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Array copy from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Vector Add from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Parallel acceleration from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Matrix multiplication from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Exec with resource from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Torch jitload_forward function from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function Torch SGEMM from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function OpenCV Generic from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function TensorFlow Lite session load from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function TensorFlow Lite session run from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Registered function TensorFlow Lite session delete from plugin helloworld
2025.01.09-16:07:18.11 - <debug> Loaded plugin helloworld from ./lib/x86_64-linux-gnu/libvaccel-helloworld.so
2025.01.09-16:07:18.11 - <debug> New rundir for session 1: /run/user/1009/vaccel/jRcbta/session.1
2025.01.09-16:07:18.11 - <debug> Initialized session 1
Initialized session with id: 1
2025.01.09-16:07:18.11 - <debug> session:1 Looking for plugin implementing noop
2025.01.09-16:07:18.11 - <debug> Returning func from hint plugin helloworld
2025.01.09-16:07:18.11 - <debug> Found implementation in helloworld plugin
2025.01.09-16:07:18.11 - <debug> [helloworld] Calling vaccel-helloworld for session 1
2025.01.09-16:07:18.11 - <debug> Released session 1
2025.01.09-16:07:18.11 - <debug> Shutting down vAccel
2025.01.09-16:07:18.11 - <debug> Cleaning up plugins
2025.01.09-16:07:18.11 - <debug> Unregistered plugin helloworld
```

### Takeaway

We made it! We just added our own plugin to implement a vAccel operation: 
- we integrated it into the vAccel source tree, 
- we built it and 
- used it to run the `noop` operation.
