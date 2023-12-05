---
layout: post
title: "Linux Virtual Filesystem"
subtitle: "An overview of Linux Virtual Filesystem"
date: 2023-12-05
author: "Michael Xi"
header-img: "img/linux-fs.jpg"
tags: [Linux, Unix, System]
---
# Virtual Filesystem Intro

The Virtual File System (VFS) is an ***abstraction layer in the kernel*** of an operating system that allows it to interact with different types of file systems in a uniform way.


> The VFS acts as an interface between the system kernel and the various file systems, enabling the kernel to treat different file systems—whether they are network file systems, disk-based file systems, or pseudo file systems—in the same manner.
> 

**VFS Design**: VFS is designed to be **flexible** and **modular**, allowing new file systems to be added easily. As new storage and network technologies are developed, VFS ensures that the operating system can interact with them. Through a **common API**, VFS provides consistent system calls for file operations such as open, read, write, and close, no matter the file system type.

VFS enables the "**everything is a file**" abstraction in Unix-like systems.

![1.png](https://s2.loli.net/2023/12/05/AOwGyghjW9I2f1i.png)

## Filesystem Interface

> The VFS is the glue that enables system calls such as `open()`, `read()`, and `write()` to work regardless of the filesystem or underlying physical medium. Such a generic interface is feasible only because the kernel implements a (Filesystem) **abstraction layer around its low-level filesystem interface**.
> 

**Example**: `ret = write (fd, buf, len);`

On one side of the system call is the generic VFS interface, providing the front end to the user-space; on the other side of the system call is the filesystem-specific backend, dealing with the implementation details.

![2.png](https://s2.loli.net/2023/12/05/vzbdVhnDEjGilI5.png)

## **Unix Filesystems Concepts**


Unix has provided four basic filesystem-related **abstractions**: `files`, `directory entries`, `inodes`, and `mount points`.

1. A ***file system*** is a **hierarchical storage of data adhering to a specific structure**. File systems contain files, directories, and associated control information. Typical operations performed on file systems are creation, deletion, and mounting.

2. A ***file*** is an **ordered string of bytes**. The first byte marks the beginning of the file, and the last byte marks the end of the file. Each file is assigned a human-readable name for identification by both the system and the user. Typical file operations include reading, writing, creating, and deleting.

3. A ***namespace*** is a container or a folder that holds various **identifiers** such as variables, functions, classes, and other namespaces. It is a way to implement scope. In Unix, each mount point gets added to the namespace and each mounted filesystem appears as part of a single unified directory tree.
   1. When you mount a filesystem in Unix, it's integrated into the directory tree at a specific point (mount point). This incorporation is handled through the namespace mechanism of the operating system.

   2. This means that irrespective of where the physical data resides (different disks, partitions, or even network locations), they appear under a single, coherent file system structure to the user and the applications.

4. The ***global hierarchy*** is the file structure of the Unix system. Unix systems organize files in a hierarchical filesystem. The top of this hierarchy is known as the root directory, represented as `/`. This hierarchical structure is sometimes also called the directory tree.

5. A ***mount point*** is a directory in the filesystem where additional information is mounted. When a filesystem is mounted on a mount point, the contents of the filesystem appear in that directory.
	> In Unix, filesystems are mounted at a specific **mount point** in a **global hierarchy** known as a `namespace`. This enables all mounted filesystems to appear as entries in a single tree. Different filesystems (which could even be on different devices or remote locations) are attached to specific directories in the Unix file hierarchy.
6. Files are organized in directories. A ***directory*** is analogous to a folder and usually contains related files. Directories can also contain other directories, called ***subdirectories***. In this way, directories may be nested to form ***paths***. Each component of a path is called a ***directory entry***.
	> A **path** example is `/home/wolfman/butter`—the root directory `/`, the directories `home` and `wolfman`, and the file `butter` are all directory entries, called ***dentries***.
	
	- In Unix, ***directories* are actually normal files that simply list the files contained therein**.

7. **Inode**: Unix systems separate the concept of a file from any associated information about it, such as access permissions, size, owner, creation time, and so on. This information is sometimes called ***file metadata*** (that is, data about the file's data) and is stored in a separate data structure from the file, called the ***inode*** (index node).

8. **Superblock**: All information is tied together with the file system's own control information, which is stored in the ***superblock***. The superblock is a data structure containing information about the file system as a whole.

# VFS Objects and Data Structures

The VFS is **object-oriented**. The structures contain both data and pointers to filesystem-implemented functions that operate on the data.


> Note that *objects* refer to **structures**—not explicit class types
> 

The four ***primary* object** types of the VFS are:

- The ***`superblock`*** object, which represents a specific mounted filesystem.
- The ***`inode`*** object, which represents a specific file.
- The ***`dentry`*** object, which represents a directory entry, which is a single component of a path.
- The ***`file`*** object, which represents an open file associated with a process.

***`operations object`*** is contained within each of these primary objects. These objects describe the methods that the kernel invokes against the primary objects.

> The operations objects are implemented as a structure of pointers to functions that operate on the parent object.
> 
- The ***`super_operations`*** object contains methods that the kernel can invoke on a specific **filesystem**, such as `write_inode()` and `sync_fs()`.
- The ***`inode_operations`*** object contains methods that the kernel can invoke on a specific **`file`**, such as `create()` and `link()`.
- The ***`dentry_operations`*** object contains methods that the kernel can invoke on a specific **directory entry**, such as `d_compare()` and `d_delete()`.
- The ***`file_operations`*** object contains methods that a process can invoke on an **`open file`**, such as `read()` and `write()`.

VFS loves **structures**. It has more than the primary objects mentioned before. Each registered filesystem is represented by a `file_system_type` structure, describing the filesystem and its capabilities. Each mount point is represented by `vfsmount` structure, containing information such as location and mount flags.

## **The Superblock Object**

The superblock object is implemented by each filesystem and is used to store information describing that specific filesystem. This object usually corresponds to the ***filesystem superblock*** or the ***filesystem control block***, which is stored in a special sector on disk.

> `struct super_block` defined in `<linux/fs.h>`
> 
> 
> ![3.png](https://s2.loli.net/2023/12/05/6i3slKceZapRwzC.png)
> 
> ![4.png](https://s2.loli.net/2023/12/05/Smd6tY8uxlfaNKE.png)
> 

### **Superblock Operation**

The most important item in the superblock object is `s_op`, which is a **pointer to the superblock operations table**. Each item in this structure is a pointer to a function that operates on a superblock object.


![5.png](https://s2.loli.net/2023/12/05/O3WbzMrJgYEN8VB.png)

![6.png](https://s2.loli.net/2023/12/05/b9Z7tjuWeNADd6l.png)

Due to the lack of object-oriented support in C, there is no way for the method to easily obtain its parent, so you have to pass it. An example of the same code between C and C++:

```c
sb->s_op->write_super(sb); // C
sb.write_super();          // C++
```

## **The Inode Object**

The inode object represents all the information needed by the kernel to manipulate a file or directory.

> `struct inode` and is defined in `<linux/fs.h>`
> 
> 
> ![7.png](https://s2.loli.net/2023/12/05/T7omEOwFNdPuHYI.png)
> 
> ![8.png](https://s2.loli.net/2023/12/05/uMrmysOJbtqlWkQ.png)
> 

### Inode Operation

**inode_operations** describes the filesystem’s implemented functions that the VFS can invoke on an inode.


> An invoke example is: `i->i_op->truncate(i)`
> 
> - `i` is a reference to a particular inode

> `inode_operations` defined in `<linux/fs.h>`
> 
> 
> ![9.png](https://s2.loli.net/2023/12/05/gBn6hyZmqiuo8AC.png)
> 
> ![10.png](https://s2.loli.net/2023/12/05/ohjKucgJmObFXRL.png)
> 

## The Dentry Object

> **Context**: VFS treats directories as a type of file. In the path `/bin/vi`, both `bin` and `vi` are files - `bin` being the special directory file and `vi` being a regular file. Despite this useful unification, VFS often needs to perform directory-specific operations, such as path name lookup. Path name lookup involves translating each component of a path, ensuring it is valid, and following it to the next component.
> 

To facilitate this, the VFS employs the concept of a directory entry (dentry). A ***dentry*** is a specific component in a path.


> Using the previous example, `/`, `bin`, and `vi` are all dentry objects. The first two are directories, and the last is a regular file. This is an important point: Dentry objects are *all* components in a path, including files. Resolving a path and walking its components is a nontrivial exercise, dentry object makes the whole process easier.
> 

> `struct dentry` defined in `<linux/dcache.h>`
> 
> 
> ![11.png](https://s2.loli.net/2023/12/05/p1ot4m3yOGkvKRZ.png)
> 
- Unlike the previous two objects, the dentry object does not correspond to any sort of on-disk data structure. Dentry object is not physically stored on the disk.
- TheVFS creates it on-the-fly from a string representation of a path name.

### Dentry State

A **dentry** object can be in one of three states: `used`, `unused`, or `negative`

1. `used`: A used dentry corresponds to a **valid inode** (`d_inode` points to an associated inode) and indicates that there are one or more users of the object (`d_count` is **positive**). A used dentry is in use by the VFS and points to valid data, and thus cannot be discarded.
2. `unused`: An unused dentry refers to a **valid inode**(`d_inode` pointing to an inode), but the VFS is not currently using the dentry object (`d_count` is **zero**). Dentry object is kept around as a **cache** in case it is needed again. 
   
    > By caching the dentry instead of destroying it early, the dentry won't need re-creation if it's needed in the future. This speeds up path name lookups. If memory needs to be reclaimed, the dentry can be discarded because it's not in use.
    > 
3. `negative`: **not** associated with a **valid inode** (`d_inode` is NULL) because either the inode was deleted or the path name was never correct to begin with. The dentry is **kept around**, however, so that future lookups are resolved quickly.
   
    > Consider a daemon that tries to read a missing config file, causing costly failed lookups. Caching these negative results can be worthwhile.
    > 

### Dentry Cache

After the VFS layer resolves each path element into a dentry object, it will caches it in the dentry cache (**dcache**) to avoid wasting effort.

The dentry cache consists of **three parts**:

1. Lists of “**used**” dentries linked off their associated inode via the i_dentry field of the inode object.
2. A doubly linked “**least recently used**” list of unused and negative dentry objects.
   
    > The list is inserted at the head, such that entries toward the head of the list are newer than entries toward the tail. When the kernel must remove entries to reclaim memory, the entries are removed from tail; those are the oldest and presumably have the least chance of being used in the near future.
    > 
3. A hash table and hashing function used to quickly resolve a given path into the associated dentry object.
   
    > Hash table is represented by the dentry_hashtable array. Each element is a pointer to a list of dentries that hash to the same value.
    > 

---

**Example**: Assume that you are editing a source file in your home directory, `/home/dracula/src/the_sun_sucks.c`. Each time this file is accessed (for example, when you first open it), the VFS must follow each directory entry to resolve the full path:`/`,`home`,`dracula`,`src`,and finally `the_sun_sucks.c`.To avoid this time-consuming operation each time this path name is accessed, the VFS can first try to look up the path name in the dentry cache. If the lookup succeeds, the required final dentry object is obtained without serious effort. Conversely, the VFS must manually resolve the path by walking the filesystem for each component of the path. After this task is completed,the kernel adds the dentry objects to the dcache to speed up any future lookups.

### Dentry Operation

> `dentry_operations` structure is defined in `<linux/dcache.h>`
> 
> 
> ![12.png](https://s2.loli.net/2023/12/05/wMArtjxmZfUQ9cG.png)
> 

## **The File Object**

The file object is used to represent a **file opened by a process**. The object points back to the dentry (which in turn points back to the inode) that actually represents the open file. Note that the file object does not actually correspond to any on-disk data like dentry object.

> The file object is the **in-memory representation of an open file**.The object (but not the physical file) is created in response to the `open()` system call and destroyed in response to the `close()` system call. Because multiple processes can open and manipulate a file at the same time, **there can be multiple file objects in existence for the same file**.
> 

> `struct file` and is defined in `<linux/fs.h>`
> 
> 
> ![13.png](https://s2.loli.net/2023/12/05/DCgWFcqI7GU3K8S.png)
> 
> ![14.png](https://s2.loli.net/2023/12/05/xcNSDEBqoH2YU7n.png)
> 

### File Operations

The operations associated with `struct file` are the familiar system calls that form the **basis of the standard Unix system calls**.

![15.png](https://s2.loli.net/2023/12/05/CHBjezvpFbnGmow.png)

![16.png](https://s2.loli.net/2023/12/05/3GhezS15siNI2FA.png)

**Some Important Operations**:

![17.png](https://s2.loli.net/2023/12/05/iU3X85vB4OHTalf.png)

![18.png](https://s2.loli.net/2023/12/05/os7rJ8beGmkCRu6.png)

![19.png](https://s2.loli.net/2023/12/05/rz3EK5PC6ngRTqM.png)

![20.png](https://s2.loli.net/2023/12/05/yDMNZ2gzIp6CiQ3.png)

![21.png](https://s2.loli.net/2023/12/05/CxP23sniyLSqJVR.png)

# **Data Structures Associated with Filesystems**

In addition to fundamental VFS objects, the kernel also uses standard data structures to manage filesystem data. One structure describes a specific variant of a filesystem (e.g. ext3, ext4, UDF), while the other describes a mounted instance of a filesystem.

## file_system_type

Linux supports many different filesystems, so the kernel requires a special structure to **describe the capabilities and behavior of each filesystem**.

> `file_system_type` is defined in `<linux/fs.h>`
> 
> 
> ![22.png](https://s2.loli.net/2023/12/05/fWKZsYi7BbX6vlT.png)
> 
- `get_sb()` function reads the superblock from the disk and populates the superblock object when the filesystem is loaded.

## vfsmount

Things get more interesting when the file system is actually mounted. At this point, the `vfsmount` structure is created. This structure **represents a specific instance of a file system**, in other words, a **mount point**.

> `vfsmount` structure is defined in `<linux/mount.h>`
> 
> 
> ![23.png](https://s2.loli.net/2023/12/05/Soh5z9tOMU8vWIp.png)
> 
> ![24.png](https://s2.loli.net/2023/12/05/5JalLjsot9BRmiG.png)
> 
- It’s **complicated** to maintain the relation between the filesystem and all the other mount points. The various **linked lists** in `vfsmount` keep track of this information.
- `mnt_flags` specifies the standard mount flags
  
    > These flags are useful on removable devices that the admin does not trust.
    > 
    > 
    > ![25.png](https://s2.loli.net/2023/12/05/eOYQSBkCymwgozn.png)
    > 

# **Data Structures Associated with a Process**

Each process on the system maintains its own list of open files, root filesystem, current working directory, and mount points. Three data structures connect the VFS layer and the processes on the system: `files_struct`, `fs_struct`, and `namespace`.

1. **files_struct** is used by the Linux kernel to keep track of all of the open files for a process.
   
    ![26.png](https://s2.loli.net/2023/12/05/7GpZfnaoKLzEXOb.png)
    
    - The array `fd_array` points to the list of open file objects. `NR_OPEN_DEFAULT` is 64, but you can adjust this macro as needed.
2. **fs_struct** contains filesystem information related to a process and is pointed at by the fs field in the process descriptor.
   
    ![27.png](https://s2.loli.net/2023/12/05/P4OxQN5Bma9Hgzb.png)
    
3. **namespace** enable each process to have a unique view of the mounted filesystems on the system—not just a unique root directory, but an entirely unique filesystem hierarchy.
   
    ![28.png](https://s2.loli.net/2023/12/05/QGMNDxSI3uHagE5.png)
    
    ![29.png](https://s2.loli.net/2023/12/05/lmH7xRu6ejdzg1A.png)
    
    - The `list` member specifies a doubly linked list of the mounted filesystems that make up the namespace.

These data structures are linked from each **process descriptor**. For most processes, the process descriptor points to unique `files_struct` and `fs_struct` structures, except for the processes created with the **clone** flag `CLONE_FILES` or `CLONE_FS`.

The `namespace` structure works in reverse. By default, all processes share the same namespace, which means that they all see the same filesystem hierarchy from the same mount table. The process is only given a unique copy of the namespace structure when the `CLONE_NEWNS` flag is specified during `clone()`. However, most processes do ***not*** provide this flag, which means that all processes inherit their parent's namespaces. Therefore, on many systems, there is only one namespace, although the functionality is only one `CLONE_NEWNS` flag away.

# Conclusion

Linux supports a wide range of file systems, from native file systems such as ext3 and ext4, to networked file systems such as NFS and Coda. There are more than 60 file systems alone in the official kernel.

The VFS layer provides these disparate file systems with both a framework for their implementation and an interface for working with the standard system calls. The VFS layer makes it easy to implement new file systems in Linux and enables those file systems to automatically interoperate via the standard Unix system calls.