You might think: what do Linux permissions and capabilities have anything to do with docker? Truth is, every container is, in fact, just a running process. A containerized process, just as any other Linux process, uses system calls, and needs permissions. And that is why all the traditional Linux stuff like system calls, namespaces, control groups, permissions, capabilities are all related to docker. It’s going to be helpful to understand some of the fundamental features of Linux so that we can see how they affect security, and in particular how they apply to containers. 

## Userspace, Kernel, and Syscalls
Applications run in what’s called user space, which has a lower level of privilege than the operating system kernel. If an application wants to do something like accessing a file, communicate using a network, create a file, or even find the time of day, it has to ask the kernel to do it on the application’s behalf. The kernel does it for you then hands control back to userspace. The programmatic interface that the userspace code uses to make these requests of the kernel is known as the system call or syscall interface.

As the name suggests, syscalls are system calls, and they’re the way that you can make requests from user space into the Linux kernel.

There is a common mechanism for making all system calls transition into the kernel, which is handled by the `libc` library. Userspace code sets up some registers including an ID of the system call it wants to make and any parameters it needs to pass to the system call. It triggers a “trap” to transition control to the kernel. That’s how userspace code makes requests of the kernel.


Linux also has pseudo filesystems that allow the kernel to communicate information to userspace. The contents look like ordinary directories and files.

The `/proc` directory is a great example. Look inside, and you’ll find all sorts of interesting information about the processes running on a machine. In some cases, like cgroups (control groups), user-space can configure parameters by writing into files under these pseudo filesystems.

It’s particularly interesting when you’re using containers because the host’s /proc holds information about all the containerized processes. This includes environment variables, which are also stored in the /proc pseudo-filesystem, meaning that your host machine has access to the environment for all your running containers. This potentially has security consequences if you’re passing secrets like certificates or database passwords into your containers through environment variables.

Many programmers working on normal applications may not feel that they’re using syscalls very often. Application developers rarely, if ever, need to worry about system calls directly, as they are usually wrapped in higher-level programming abstractions. The lowest-level abstraction you’re likely to come across as an app developer is the `glibc` library or the Golang syscall package.

In practice, however, we use syscalls a lot, because even everyday activities like making files or changing directories involve syscalls on Linux.
There are some 300+ different system calls, with the number varying according to the version of the Linux kernel. Here are a few examples:

- read: read data from a file
- write: write data to a file
- open: open a file for subsequent reading or writing
- execve: run an executable program
- chown: change the owner of a file
- clone: create a new process

## Permissions

On Linux, any Linux system, whether you are using docker containers or not, you are using files. Everything is a file in Linux. Not only documents but also hardware like disks, keyboards, monitors, even network communications, are all exposed through the filesystem. And that is why permission is the cornerstone of security.

### setuid

Normally, when you execute a file, the process that gets started inherits your user ID. If the file has the setuid bit set, the process will have the user ID of the file’s owner. Let’s do an experiment with the command sleep, but in order to not change the default sleep, let’s copy it first:


```
$ cp /usr/bin/sleep ./my-sleep
$ ll ./my-sleep
-rwxr-xr-x 1 zhan4429 itap 33128 Aug  8 09:54 ./my-sleep
```
Now let’s set the setuid bit for the my-sleep:
```
$ chmod +s ./my-sleep
$ ll ./my-sleep
-rwsr-sr-x 1 zhan4429 itap 33128 Aug  8 09:54 ./my-sleep
```

We do the experiment by running:
```
$ sudo ./my-sleep 100
```
In another terminal, we see the running process again:

```
vagrant@vagrant ~ $ ps ajf
 PPID   PID  PGID   SID TTY      TPGID STAT   UID   TIME COMMAND
19286 19287 19287 19287 pts/1    19408 Ss    1000   0:00 -bash
19287 19408 19408 19287 pts/1    19408 R+    1000   0:00  \_ ps ajf
18406 18407 18407 18407 pts/0    19405 Ss    1000   0:00 -bash
18407 19405 19405 18407 pts/0    19405 S+       0   0:00  \_ sudo ./my-sleep 100
19405 19406 19405 18407 pts/0    19405 S+    1000   0:00      \_ ./my-sleep 100
    1   960   960   960 tty1       960 Ss+      0   0:00 /sbin/agetty -o -p -- \u --noclear tty1 linux

```

Now as you can see, the sudo is running as UID 0 but the my-sleep command is running as UID 1000 which is the UID of the vagrant user, meaning the my-sleep command didn't start with the user who runs it but started with the file owner.

## Capabilities
For the purpose of performing permission checks, traditional UNIX implementations distinguish two categories of processes: privileged processes (whose effective user ID is 0, referred to as superuser or root), and unprivileged processes (whose effective UID is nonzero). Privileged processes bypass all kernel permission checks, while unprivileged processes are subject to full permission checking based on the process’s credentials (usually: effective UID, effective GID, and supplementary group list).
Starting with kernel 2.2, Linux divides the privileges traditionally associated with superuser into distinct units, known as capabilities, which can be independently enabled and disabled. Capabilities are a per-thread attribute.

### History
So, what is before kernel 2.2?
Before capabilities, we only had the binary system of privileged and non-privileged processes, which means, either your process can do everything (admin level syscalls), or it was restricted like a normal user. to the subset of a standard user.
If you want to run some program (like we tested above, the ping) by normal users but it requires privileged syscalls, we must use the setuid bit. However, this literally grants our “my-ping” all privileged access. Since software has bugs, these executables that have setuid bit are the primary targets for hackers: if you can find a bug in ping, you can become root.
This wasn’t a great situation of course, and based on today’s modern cloud security situation, you would guess what came after that: that’s right, “RBAC”.
The idea is simple: split all the possible privileged kernel calls into smaller groups (roles), and only grant access to a certain thread when needed. This is the history of Linux capabilities.

## Namespaces
Namespaces are one of a feature in the Linux Kernel and fundamental aspect of containers on Linux. On the other hand, namespaces provide a layer of isolation. Docker uses namespaces of various kinds to provide the isolation that containers need in order to remain portable and refrain from affecting the remainder of the host system. Each aspect of a container runs in a separate namespace and its access is limited to that namespace.

## Cgroups
Cgroups provide resource limitation and reporting capability within the container space. They allow granular control over what host resources are allocated to the containers and when they are allocated.

Common control groups
- CPU
- Memory
- Network Bandwidth
- Disk
- Priority





