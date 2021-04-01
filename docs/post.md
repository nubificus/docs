# Add a simple vAccel plugin

vAccel is a simple yet powerful framework that semantically exposes function prototypes to the user, while executing the relevant operation under the hood. vAccel is structured in two main components: the runtime system, vAccelRT and the plugins. Plugins expose one or more functionalities to the upper-layers of the runtime system, in a way that the user can directly consume these functions without caring about the underlying implementation. 

In this short tutorial, we will go through adding a simple vector add operation
to vAccelRT, and build a plugin that implements this operation using OpenCL.
The purpose of this tutorial is to showcase the simplicity of adding an
operation and how straightforward adding a new plugin to support an operation
is.

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

Create a build directory and prepare the relevant files for building (including just the examples):

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

Lets move on to something a bit more complicated than a `noop`.


## Add an operation

Suppose we have some code written for a project and we want to integrate it in
vAccel, in order to expose the functionality to users without them caring about
the underlying implementation. 

In short, lets say we have a `vector add` operation, in `OpenCL`. A first-page google result for `opencl example vector add` showed the following github repo: `https://github.com/mantiuk/opencl_examples` which we forked at `https://github.com/nubificus/opencl_examples`.

To build, you will need a working OpenCL installation. On my debian-based OS I just did `apt-get install libpocl-dev`.

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

So, if we want to expose this `vector add` operation via vAccelRT, we need to do two things:

- expose the function prototype as a vAccel operation, in this case `int vector_add()`
- transform this code into a vAccel plugin.

Lets see how easy it is to do both!

### expose the `vector add` operation

To add a vAccel operation we need to:

- add it as an operation in `vaccel_ops.h`
- add its plugin matching code to `vaccel_ops.c`

Specifically the patch to add this operation is the following:

```
diff --git a/src/vaccel_ops.h b/src/vaccel_ops.h
index 0ca98e6..9e1a81f 100644
--- a/src/vaccel_ops.h
+++ b/src/vaccel_ops.h
@@ -9,7 +9,8 @@
 #define VACCEL_IMG_CLASS    2
 #define VACCEL_IMG_DETEC    3
 #define VACCEL_IMG_SEGME    4
-#define VACCEL_FUNCTIONS_NR 5
+#define VACCEL_VADD        5
+#define VACCEL_FUNCTIONS_NR 6
 
 static const char *vaccel_op_name[] = {
        "noop",
@@ -17,6 +18,7 @@ static const char *vaccel_op_name[] = {
        "image-classification",
        "image-detection",
        "image-segmentation",
+       "vadd",
 };
 
 inline static const char *vaccel_op_type_str(uint8_t op_type)
@@ -44,4 +46,6 @@ int vaccel_image_segmentation(struct vaccel_session *sess, void *img,
                unsigned char *out_imgname, size_t len_img,
                size_t len_out_imgname);
 
+int vaccel_vector_add(struct vaccel_session *sess);
+
 #endif /* __VACCEL_OPS_H__ */
diff --git a/src/vaccel_ops.c b/src/vaccel_ops.c
index f7d80b9..76b5cf0 100644
--- a/src/vaccel_ops.c
+++ b/src/vaccel_ops.c
@@ -10,6 +10,7 @@ typedef typeof(vaccel_sgemm) sgemm_t;
 typedef typeof(vaccel_image_classification) image_classification_t;
 typedef typeof(vaccel_image_detection) image_detection_t;
 typedef typeof(vaccel_image_segmentation) image_segmentation_t;
+typedef typeof(vaccel_vector_add) vadd_t;
 
 int vaccel_noop(struct vaccel_session *sess)
 {
@@ -45,3 +46,19 @@ int vaccel_image_classification(struct vaccel_session *sess, const void *img,
        return plugin_op(sess, img, out_text, out_imgname, len_img,
                        len_out_text, len_out_imgname);
 }
+
+int vaccel_vector_add(struct vaccel_session *sess)
+{
+       if (!sess)
+               return VACCEL_EINVAL;
+
+       vaccel_debug("session:%u Looking for plugin implementing vector add",
+                       sess->session_id);
+
+       //Get implementation
+       vadd_t *plugin_op = get_plugin_op(VACCEL_VADD);
+       if (!plugin_op)
+               return VACCEL_ENOTSUP;
+
+       return plugin_op(sess);
+}

```
Let's go through what the code does. 

XXXXXXX TBC
XXXXXXX


### Transform our code to a vAccel plugin

To integrate our code to vAccel as a plugin we need to:

- build it as a shared library and replace `main` with the name of our
  choosing; in this case `vector_add`,
- create the glue code to register this operation/symbol with `VACCEL_VADD`.

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

Our work is not done though; we need to create a vAccel plugin and register it
as one of the implementations of the `VACCEL_VADD` operation we added above.

Let's use the NOOP plugin as reference; we copy its contents to a new
directory, called `vadd`. We go back to the vAccelRT base source dir and
execute the following:

```
cp -avf plugins/noop plugins/vadd
```

we get rid of any noop mentions in there so we come up with the following directory structure:

```
plugins/vadd
├── CMakeLists.txt
└── vaccel.c
```

and the following contents for CMakeLists.txt:

```
set(include_dirs ${CMAKE_SOURCE_DIR}/src)
set(SOURCES vaccel.c ${include_dirs}/vaccel.h ${include_dirs}/plugin.h)

add_library(vaccel-vadd SHARED ${SOURCES})
target_include_directories(vaccel-vadd PRIVATE ${include_dirs})

set(VADD_LIB "/home/ananos/develop/opencl_examples/build/vector_add")
message("Using VADD_LIB ${VADD_LIB}")

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

struct vaccel_op op = VACCEL_OP_INIT(op, VACCEL_VADD, ocl_vector_add);

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

