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
  c_args : plugins_c_args,
  dependencies : libvaccel_dep,
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
	VACCEL_OP_INIT(ops[0], VACCEL_NOOP, helloworld_helloworld),
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
  description : 'Build the helloworld plugin')
```

Now we can build our new plugin, by specifying the new option we just added:

```
meson setup -Dplugin-helloworld=enabled build --reconfigure
meson compile -C build
meson install -C build
```

Under the `<path>` directory we see that a new plugin is now available, `libvaccel-helloworld.so`. Lets use this one instead of the `noop` one!

```
LD_LIBRARY_PATH=./lib/x86_64-linux-gnu/ VACCEL_LOG_LEVEL=4 VACCEL_PLUGINS=./lib/x86_64-linux-gnu/libvaccel-helloworld.so ./bin/noop
```
should return:
```
2025.03.23-11:47:27.16 - <debug> Initializing vAccel
2025.03.23-11:47:27.16 - <info> vAccel 0.6.1-194-19056528-dirty
2025.03.23-11:47:27.16 - <debug> Config:
2025.03.23-11:47:27.16 - <debug>   plugins = ./lib/x86_64-linux-gnu/libvaccel-helloworld.so
2025.03.23-11:47:27.16 - <debug>   log_level = debug
2025.03.23-11:47:27.16 - <debug>   log_file = (null)
2025.03.23-11:47:27.16 - <debug>   profiling_enabled = false
2025.03.23-11:47:27.16 - <debug>   version_ignore = false
2025.03.23-11:47:27.16 - <debug> Created top-level rundir: /run/user/1008/vaccel/Ge89tr
2025.03.23-11:47:27.16 - <info> Registered plugin helloworld 0.6.1-194-19056528-dirty
2025.03.23-11:47:27.16 - <debug> Registered op noop from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op blas_sgemm from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op image_classify from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op image_detect from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op image_segment from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op image_pose from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op image_depth from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op exec from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op tf_session_load from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op tf_session_run from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op tf_session_delete from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op minmax from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op fpga_arraycopy from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op fpga_vectoradd from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op fpga_parallel from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op fpga_mmult from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op exec_with_resource from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op torch_jitload_forward from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op torch_sgemm from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op opencv from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op tflite_session_load from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op tflite_session_run from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Registered op tflite_session_delete from plugin helloworld
2025.03.23-11:47:27.16 - <debug> Loaded plugin helloworld from ./lib/x86_64-linux-gnu/libvaccel-helloworld.so
2025.03.23-11:47:27.16 - <debug> New rundir for session 1: /run/user/1008/vaccel/Ge89tr/session.1
2025.03.23-11:47:27.16 - <debug> Initialized session 1
Initialized session with id: 1
2025.03.23-11:47:27.16 - <debug> session:1 Looking for plugin implementing noop
2025.03.23-11:47:27.16 - <debug> Returning func from hint plugin helloworld
2025.03.23-11:47:27.16 - <debug> Found implementation in helloworld plugin
2025.03.23-11:47:27.16 - <debug> [helloworld] Calling no-op for session 1
2025.03.23-11:47:27.16 - <debug> Released session 1
2025.03.23-11:47:27.16 - <debug> Cleaning up vAccel
2025.03.23-11:47:27.16 - <debug> Cleaning up sessions
2025.03.23-11:47:27.16 - <debug> Cleaning up resources
2025.03.23-11:47:27.16 - <debug> Cleaning up plugins
2025.03.23-11:47:27.16 - <debug> Unregistered plugin helloworld
```

### Takeaway

We made it! We just added our own plugin to implement a vAccel operation: 
- we integrated it into the vAccel source tree, 
- we built it and 
- used it to run the `noop` operation.
