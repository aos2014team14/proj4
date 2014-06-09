#The Linux File System (`ext2`)
*Seoul National University* / *Advanced Operating Systems* / *2014 Spring Semester* / *Project 4*

*Kang Min Yoo* / *Kang Hyeonsu* / *Camilo A. Celis Guzman*


##Faster testing
After losing our minds while waiting for Android `arm` architecture to emulate, we decided to implement and test everything using a different architecture first, `x86`, and then port it to `arm`. Working with `x86` decreased by 10 times the testing time due to short time it takes to emulate. In order to do this we needed another set of tools and heres how we did it. 

###How to Test the Code in x86

Download the following repositories by running the commands:

	git clone https://android.googlesource.com/platform/external/qemu.git/
	git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/x86/i686-linux-android-4.7/

Add the appropriate paths to `PATH` variable. Open `/etc/profile` in root mode first.

	sudo gedit /etc/profile

Add the following lines at the end of the file. Note that you need to specify the path to the repositories you have just cloned.

	ANDROID_TOOLCHAIN=<path-to-qemu>/distrib/kernel-toolchain/
	ANDROID_i686_TOOLCHAIN=<path-to-i686-linux-android-4.7>/bin/
	export ANDROID_TOOLCHAIN
	export ANDROID_i686_TOOLCHAIN
	export PATH=$PATH:$ANDROID_TOOLCHAIN:$ANDROID_i686_TOOLCHAIN

Run the following line to apply the changes immediately.

	source /etc/profile

That's it. Run `./build x86` to build, or `./emulate <avd-name> x86` to emulate. Make sure that you have `emulator64-x86` in `PATH`.

#About
As mentioned before, but it is important to notice how many location aware applications are there these days. In this project, we have embedded the location information into the Linux file system on which Android is based on. We used the native, device-dependent support for the storage of GPS information. 

#Design
There are several steps we need to take in order to embed extra information as the location into the file system. We modify the ext2 file system's physical representation on disk; so we used the ext2 file system utilities in e2fsprogs.  We first have modified the ext2 virtual file system related `inode_operations` structures, and then added a structure inside of the `ext2_inode` structure to keep the location information each time a file is created or modified on the physical disk.

##GPS Location Data Structure##
Each GPS Location is represented by integers rather than floating numbers because of some issues in handling floating-points in linux kernel. We have added some more details on the issue [below](#problems). The following is the declaration of such representation.

```
struct _gps_location {
	__u64 latitude;
	__u64 longitude;
	__u32 accuracy;
};
```

When a user program sets a new gps location using the system call, floating numbers from `struct gps_location` are converted using built-from-scratch converters to integers with precision up to 2^-20 of the original floats. For `accuracy`, the precision is up to 2^-4. For example, a GPS location of (132.222312, 54.139863, 19.23) is converted to (138645143, 56769761, 308). In terms of units, those integers are in units of 11 cm for latitude and longitude and 6.25 cm for accuracy (assuming that the distance between two coordinates is the length of the line that directly connects them). The method we used to convert `double` to `long` without using cpu instructions is mentioned in the [Problems](#problems) section. When checking the overlapping of two GPS locations, these factors are considered with care.

##Physical Reprentation##
We added a `gpsloc` structure inside of the `ext2_inode` structure:

```C
struct ext2_inode {
	...
 	struct {
  		__u64 i_latitude; /* AOS Project 4 */
  		__u64 i_longitude;
  		__u32 i_accuracy;
		__u32 i_coord_age;
		__u8  i_is_set;
  	} gpsloc;
};
```
as one can see, we use the first three variabls inside of the structure as to keep the gps location values and its accuracy, and keep the data age.

Another additional field, `i_is_set`, is used to determine whether gps_location was set on the inode before or not. This is used for sanity check, and it is crucial in preventing glitches.

Above physical change to `ext2` block is applied to `e2fsprogs`'s `lib/ext2fs/ext2_fs.h` as well. And of course, the magic number on the same file is updated too.

##Logical Representation##
We can't keep making changes on the disk itself, so we assigned that job to the original code, and we added the following fields to `struct inode` to read and write gps-related fields directly from the memory.

```C
struct inode {
	...
	/* AOS Project 4 */
	int i_loc_set;
	struct _gps_location i_loc;
	int i_loc_age;
};
```

Reading inode's metadata from the disk is carried out by `ext2_iget` and writing back to the disk is done by `ext2_write_inode`, so we added appropriate assignments to the functions respectively.

##Adding System Calls 

For `set_gps_location`, the implementation is pretty straightforward, we just copy the argument `gps_location` strcture to the kernel space value and copy it to the global variable, `gps_location`:

```C
SYSCALL_DEFINE1(set_gps_location, struct gps_location __user *, loc)
{
    struct _gps_location kloc;

    /* Normal users which do not own the root's right cannot update gps */
    if (current_uid() != 0 && current_euid() != 0)
        return -EACCES;

    if (!access_ok(VERIFY_READ, loc, sizeof(struct gps_location)))
        return -EINVAL;

    if (copy_from_user(&kloc, loc, sizeof(struct gps_location)))
        return -EFAULT;

    print_gps_location(&kloc);

    kloc.latitude = convert_double_to_long(kloc.latitude, 20);
    kloc.longitude = convert_double_to_long(kloc.longitude, 20);
    kloc.accuracy = convert_float_to_int(kloc.accuracy, 4);

    print_gps_location(&kloc);

    write_lock(&gps_location_lock);
    gps_location = kloc;
    gps_timestamp = CURRENT_TIME_SEC.tv_sec;
    write_unlock(&gps_location_lock);

    return 0;
}

SYSCALL_DEFINE2(get_gps_location, 
                const char __user *, pathname,
                struct gps_location __user *, loc)
{
    ...
    ret = inode->i_op->get_gps_location(inode, &kloc);

    if (ret < 0)
        goto get_gps_location_end;

    kloc.latitude = convert_long_to_double(kloc.latitude, 20);
    kloc.longitude = convert_long_to_double(kloc.longitude, 20);
    kloc.accuracy = convert_int_to_float(kloc.accuracy, 4);

    ...

    get_gps_location(&cur_loc);
    if (!check_gps_overlap(&kloc, &cur_loc)) {
        ret = -EPERM;
        goto get_gps_location_end;
    }

    if (copy_to_user(loc, &kloc, sizeof(struct gps_location))) {
        ret = -EFAULT;
        goto get_gps_location_end;
    }

  get_gps_location_end:
    kfree(kpathname);
    return ret;
}
```
As for getting the gps location, we first index into the path that has been passed as an argument, then by calling `inode->i_op->get_gps_location(inode, &kloc)` we get the actual link to the corresponding ext2 file system `get_gps_location` function.

##Adding File System Operations##
We first create the links inside of the inode_operations related structures:
In `ext2/namei.c`

```C
const struct inode_operations ext2_dir_inode_operations = {
	...
	.get_gps_location = ext2_get_gps_location;
	.ext2_set_gps_location = ext2_set_gps_location;
};

const struct inode_operations ext2_special_inode_operations = {
	...
	.get_gps_location = ext2_get_gps_location;
	.ext2_set_gps_location = ext2_set_gps_location;	
};
```
In `ext2/symlink.c`

```C
const struct inode_operations ext2_symlink_inode_operations = {
	...
	.get_gps_location = ext2_get_gps_location;
	.ext2_set_gps_location = ext2_set_gps_location;
};

const struct inode_operations ext2_fast_symlink_inode_operations = {
	...
	.get_gps_location = ext2_get_gps_location;
	.ext2_set_gps_location = ext2_set_gps_location;	
};
```	
In `ext2/file.c`

```C
const struct inode_operations ext2_file_inode_operations = {
	...
	.get_gps_location = ext2_get_gps_location;
	.ext_w_set_gps_location = ext2_set_gps_location;
};
```
The actual `ext2_get_gps_location()` is defined in `ext2/inode.c`:
In `ext2/inode.c`

```C
int ext2_set_gps_location(struct inode *inode)
{
	struct gps_location cur_loc = get_gps_location();

	inode->gpsloc.i_latitude = cur_loc.latitude;
	inode->gpsloc.i_longitude = cur_loc.longitude;
	inode->gpsloc.i_accuracy = cur_loc.accuracy;
	inode->gpsloc.i_timestamp = CURRENT_TIME_SEC.tv_sec;

	return 0;
}

int ext2_get_gps_location(struct inode *inode, struct gps_location *loc)
{
	int age;

	if (unlikely(!loc))
		return -EINVAL;

	loc->latitude = (double)inode->gpsloc.i_latitude;
	loc->longitude = (double)inode->gpsloc.i_longitude;
	loc->accuracy = (double)inode->gpsloc.i_accuracy;
	age = (int)(CURRENT_TIME_SEC.tv_sec - loc->i_timestamp);

	if (age < 0)
		return -EINVAL;

	return age;
}
```
When we first create the directory cache entry for the new file, we initialize the gps location info as the information at that moment too:

```C
static int ext2_create (struct inode *dir, struct dentry * dentry, umode_t mode, struct nameidata *nd)
{
	...
	ext2_set_gps_location(dir); /* AOS Project 4 */
	
	return ext2_add_nondir(dentry, inode);
}

static int ext2_mkdir(struct inode * dir, struct dentry * dentry, umode_t mode)
{
	...
	err = ext2_set_gps_location(inode); /* AOS Project 4 */
	...
}
```
Also upon any modification we update the gps location accordingly:

```C
static int ext2_rename (struct inode * old_dir, struct dentry * old_dentry,
  	struct inode * new_dir,	struct dentry * new_dentry )
{
	...
	if (ext2_set_gps_location(new_inode)) /* AOS Project 4 */
		return -EINVAL;
	...
}
```

And this is pretty much all about what need be done to keep the right updates in gps location and time information.

##GPS Authentication##
GPS Authentication works by denying access to an ext2 file or directory if the gps location of the file or directory does not overlap with the current gps location. To deny accesses, we modified one of the functions in the routine of `open` system call. `open` syscall calls several inode related functions like `i_op->lookup()`, `audit_inode()` etc. One of the functions was very much suited to our needs (`may_open()`), so we added the GPS checking routines into the function. The following shows the code snippet.


```C
if (!(flag & O_CREAT) && inode->i_loc_set) {
	get_gps_location(&cur_loc);
	inode->i_op->get_gps_location(inode, &inode_loc);
	if (!check_gps_overlap(&cur_loc, &inode_loc))
		return -EPERM;
}	
```

The routine first checks whether `may_open` is called by a file-creating routine or a file-opening routine. Only when it is the latter case, it proceeds to check whether the `inode` has been set with GPS location before. Only then, it will check against the current GPS location and return an error if those two do not overlap. 

`check_gps_overlap()` function is defined in `gps.c`. It checks if the distance between the two locations is smaller than the sum of their accuracies. Only if the distance is smaller, then we can safely say that the two locations are recorded at one place. We could add a threshold of overlap percentage to be more precise, but for the demonstration purpose, we kept it as simple as possible.

##User Programs
We included the two required test programs under the directory `/proj`. 

1). `gpsupdate` Uses 14 different locations (`gpsloc struct`s) and RR around them sleeping `x` amount of seconds (by default it is 10 seconds, but it can be passed as an argument). 

The locations used were:

| Latitude, Longitude 		| Location Description 		|
|-------------------------------|-------------------------------|
| 37.448727, 126.952538		| SNU - Building 302		|	
| 37.450030, 126.952506		| SNU - Building 301 		|
| 37.455828, 126.954688		| SNU - Kangmin's Lab 		|
| 37.459315, 126.952413		| SNU - Central Library		|
| 37.448635, 126.950939 	| SNU - Engineering House	|
| 37.422234, -122.084037	| Google Inc.			|
| 37.331681, -122.030412	| Apple Inc.			|
| 32.149989, -110.835842	| Lot's of Airplanes		|
| 33.747252, -112.633853	| A Giant Triangle		|
| 45.123853, -123.113603	| Firefox Logo			|
| 41.303921, -81.901693 	| Heart-shaped Lake		|
| -33.350534, -71.653268	| World's Biggest Pool		|
| 33.921277, -118.391674	| Mattel Logo			|
| 34.871778, -116.834192	| Solar File 			|

![SNU - Building 302](http://imgur.com/OsYwSo3.png)
![SNU - Central Library](http://imgur.com/HDABaRw.png)
![SNU - Kangmin's lab](http://imgur.com/qUkaIhb.png)
![Google Inc.](http://imgur.com/jRuApgJ.png)
![Apple Inc.](http://imgur.com/OXtGrXs.png)
![A Giant Triangle](http://imgur.com/bL5EAuC.png)
![World's Biggest Pool](http://imgur.com/dAObm0w.png)


2). `file_loc` simply gets a `path` as parameter and check whether this file has `gpsloc` information, if so it outputs the file's gpslocation, accuracy, age, and a URL for the exact location in Google maps.

In order to run this two programs make use of the script included in the same directory: `run` that expects the program name to execute.

Also, we included various scripts to facilitate the testing process:

- `create_fs.sh` creates a new `ext2` file system (`proj4.fs`) using the `mkext2` utility.
- `setup` pushes every binary file found in the `proj/bin/` directory to `/sbin/` on the device.
- `setup_fs <arch>` sets the symlinks for loop devices, remounts the file system to writable, pushes busybox, sets up losetup, pushes and mounts the new file system on the device (assuming that the emulator is running).
- `pull_fs` creates various files and folders in the new file system, unmount and pulls it. The created directory tree is of the form:

````
|-- proj4
|   |-- dir1
|      `-- file1.txt
|      `-- file2.txt
|   |-- dir2
|      `-- file1.txt
|      `-- file2.txt
````

> `<arch>` is either `arm` or `x86

##Problems
In this project we run into various problems. The first and most cumbersome one to solve was the inability of using floating point number types inside the kernel. We were told that the `struct gps_location` couldn't be changed, and furthermore that the types couldn't be cast. At first, we were not aware of that *minor* issue and the output error were very non-intuitive so we spent a considerably amount of time figuring out and debugging for that error, but it turned out to be the lack of floating point types inside the kernel. Below is an example error log.

![alt tag](http://i.imgur.com/ucXOkIh.jpg)

We solved it by representing `struct gps_location` as integer types, which are defined in `struct _gps_location`. As we have mentioned earlier, casting `double` to `long` or `int` is problematic in ARM Linux. We could have solved it by using soft floating point representation like using `NWFPE` or `FASTFPE` module, but there was little information about enabling those module (we tried using CONFIG_FPE_NWFPE and CONFIG_FPE_FASTFPE as it was suggested in a very old documentation, but they didn't make a difference), so we had to give up on the idea. Instead, we implemented a bit-by-bit [double-precision floating-point format](http://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64) and [single-precision floating-point format](http://en.wikipedia.org/wiki/Single-precision_floating-point_format#IEEE_754_single-precision_binary_floating-point_format:_binary32) converter. Those functions are declared statically in `kernel/gps.c` as `convert_double_to_long()` and `convert_float_to_int()` vice versa.

But implementing them was not easy either. there was weird bug where the conversion seemed ok when compiled and run in the host computer but not on the emulator. It turned out that arithmetic shift operators on `long` or 64-bit variables were not supported on ARM. We do not know if this is the general but unspoken case, or if it is an Android-Linux-specific problem, but we had to overcome the problem by splitting the `long` variables into two `__u32` variables and perform segregated shift operations from there.

Another problem we faced was this error after trying to mount the file system : `ioctl LOOP_SET_FD failed: Device or resource busy`. We solved by cross compiling `busybox` and pushing the binary on the device and run the utility `losetup` to correctly set the loopback devices; however, we first needed to remount the entire `/` of the device in order to work. All the steps are summarized on the scripts in `/proj` directory.  

#Demonstration
The preset file system, which contains two files named `1.txt` and `2.txt` and a directory name `dir1`, is placed in `proj/fs/` as `proj4.fs`. Below is the screenshot of printing the coordinates of each file and directory. The last line demonstrates that the access to the file is not permitted if the current GPS location does not overlap with the GPS location stored in the file or directory.

![Demo](http://i.imgur.com/qS4SZqw.jpg)
