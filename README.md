# cvmfsexec package

Whenever possible it is best to install standard
[cvmfs](https://cernvm.cern.ch/fs/) from native OS packages, but
sometimes that is not an option.
This package is for mounting cvmfs as an unprivileged user, without the
cvmfs package being installed by a system administrator.  The package can
do this in 4 different ways:

1. On systems where only fusermount is available, the `mountrepo` and
   `umountrepo` commands can be used to mount cvmfs repositories in the
   user's own file space.  That path can then be bindmounted at /cvmfs
   by a container manager such as
   [apptainer](https://github.com/apptainer/apptainer)
   (formerly known as singularity).
   A big disadvantage compared to mode 3 below is that if the processes
   are hard-killed (kill -9), mountpoints are left behind and difficult
   to clean up.
2. On systems where fusermount is available and unprivileged user
   namespaces are enabled, but unprivileged namespace fuse mounts are not
   available (in particular RHEL <=7.7 with
   `sysctl user.max_user_namespaces` > 0),
   the `cvmfsexec` command can mount cvmfs repositories, map them into
   /cvmfs, and unmount them when it exits.  apptainer may even be
   run unprivileged from cvmfs from within cvmfsexec (it has to run
   unprivileged because setuid-root does not work inside a user
   namespace).  
   This mode shares the disadvantage that mode 1 has compared to mode 3
   in that if the processes are hard-killed, mountpoints are left behind.
3. On systems where unprivileged namespace fuse mounts are available
   (newer kernels >= 4.18 as on RHEL8 or >= 3.10.0-1127 as on RHEL 7.8),
   the `cvmfsexec` command can entirely manage mounting and unmounting
   of cvmfs repositories in the namespace, so if they get killed
   everything gets cleanly unmounted.  fusermount is not needed in this
   case.
4. On systems that have no fusermount nor unprivileged user namespace
   fuse mounts but do have a setuid installation of singularity >= 3.4
   or apptainer,
   an entirely separate command in this package `singcvmfs` can mount
   cvmfs repositories inside a container using the `--fusemount` feature.
   With singularity >= 3.6 and RHEL >= 7.8 and
   unprivileged user namespaces enabled,
   this can also be used with unprivileged singularity or apptainer.

# Supported operating systems

Operating systems currently supported by this package are Red Hat
Enterprise Linux (versions 7, 8, and 9) and its derivatives (CentOS,
Scientific Linux, Rocky Linux, Alma Linux)
and SUSE Linux Enterprise (version 15)
and its derivatives (openSUSE Leap).  All of those support the
x86_64 architecture, and RHEL8 also supports ppc64le and aarch64.
Debian/Ubuntu probably could be supported but it would require some
development in the `makedist` command.

Even though RHEL7 is now officially End of Life, cvmfsexec will still
support it for a while because some people continue to use it with
extended support.

# Making the cvmfs distribution

All of the ways this package supports unprivileged cvmfs make use of a
copy of the cvmfs software.  The cvmfs software and configuration are
expected to be in a `dist` subdirectory under where the scripts are.  The
easiest way to create the dist directory is to use `makedist`.  It takes a
parameter of `osg`, `egi`, or `default` to download the latest cvmfs and
configuration rpm from one of those three sources. Note: `egi` does not
currently provide rpms for RHEL8 or 9. Additionally, specifying `none`
will download from the `default` source but exclude the
`cvmfs-config-default` package.

By default a distribution for `cvmfsexec` and `mountrepo/umountrepo` is
created.  To instead make a distribution for `singcvmfs`, add the `-s`
makedist option.  By default the distribution made will match the 
operating system it runs on, but another distribution can be selected
with the `-m` option.  See the makedist usage for supported machine types.

To customize any cvmfs configuration settings, put them in
`dist/etc/cvmfs/default.local`.  In particular you may want to set
CVMFS_HTTP_PROXY, although the default is to use WLCG Web Proxy Auto
Discovery.  You may also want to set CVMFS_QUOTA_LIMIT, otherwise the
default is 4000 MB.  The default CVMFS_CACHE_BASE for the cache
shared between the cvmfs repository is under the dist directory,
`dist/var/lib/cvmfs`, unless the `-m` option is used to add an e2fs
filesystem (details [below](#optionally-mount-a-scratch-filesystem)).
Make sure that the cache does not get shared between multiple machines.

## Self-contained distribution

For the cases where `cvmfsexec` or `singcvmfs` can be used, you can also
make a self-contained distribution in a single file that contains both
the command and all supporting files.  This makes it easy to share the
distribution with other users or distribute it to many machines.

After running makedist and making any customizations you want, use
this to make a cvmfsexec distribution:
```
makedist -o /tmp/cvmfsexec
```
or this to make a singcvmfs distribution:
```
makedist -s -o /tmp/singcvmfs
```

Executing a cvmfsexec file that is created in that way leaves behind a
.cvmfsexec directory in the directory where it is run from, and running
a singcvmfs file leaves behind a .singcvmfs directory.

# cvmfsexec command (modes 2 and 3)

The cvmfsexec command requires unprivileged user namespaces.  On RHEL8/9
unprivileged user namepaces (and user namespace fuse mounts) are
available by default, but on RHEL7 they need to be enabled by setting a
sysctl parameter as detailed in the
[OSG unprivileged apptainer instructions](https://osg-htc.org/docs/worker-node/install-apptainer/#enabling-unprivileged-apptainer).
In addition cvmfsexec requires fusermount on kernels older than
those that come with RHEL7.8.

To execute a command in an environment where cvmfs repositories are
mounted at /cvmfs and automatically unmounted upon exit, use
`cvmfsexec repository.name ... -- [command]` where the default command
is $SHELL.  It will automatically mount the configuration repository if
one is defined.

Unless disabled with the `-N` option,
inside the command you can mount additional repositories by using
`$CVMFSMOUNT repository.name`.  Since the mounts have to happen outside
the user namespace, it actually sends a message to the original process
to mount, and makes the current process wait until completion.
Repositories that are already mounted are ignored.  You can also unmount
repositories from within the command with `$CVMFSUMOUNT repository.name`.
If you want to use this feature and also
invoke additional processes within the original process that are
not trustworthy, such as user payloads that are invoked with
`--contain` option of singularity or apptainer,
then close the $CVMFSEXEC_CMDFD file descriptor
for those processes.  This can be done in bash with
`exec {CVMFSEXEC_CMDFD}>&-`.

Note that setuid-root programs do not work inside an unprivileged user
namepace, so if you use singularity or apptainer it has to be run unprivileged.

Cache considerations: by default cvmfsexec starts a cache manager
process for all the cvmfs repositories it mounts, which means only one
cvmfsexec process can share a cache on a single machine.  The cvmfs
configuration could be set to use a different path for a cache for
different invocations (the default is `dist/var/lib/cvmfs`), or it could
be set to use cvmfs alien cache mode which doesn't use a cache manager,
but the best approach is to start cvmfsexec from a pilot process and run
only one pilot per machine.  If possible the cache should be on local
disk, because otherwise the many file accesses can overwhelm a shared
filesystem's metadata server.  Also, many network filesystems types
cannot handle the locks that cvmfs creates.
If there is no local disk the next best
option is to mount a filesystem separately for each worker
node from a big enough file on the shared filesystem, or alternatively
a RAM disk on the local node if there is sufficient RAM.
The `-m` option (described in the next section) can mount that separate
scratch filesystem for you in the cvmfsexec namespace.
Otherwise, if the cache directory needs to be changed that can be done
by setting CVMFS_CACHE_BASE in `dist/etc/cvmfs/default.local`,
or you can just install the whole distribution in the local filesystem.

## Optionally mount a scratch filesystem

If there is no local disk available, cvmfsexec can mount a scratch
ext2/3/4 filesystem with the `-m` option, using the fuse2fs command.
The filesystem will appear in the cvmfsexec namespace at `/e2fs`.
If you have not set CVMFS_CACHE_BASE in `dist/etc/cvmfs/default.local`
then this filesystem will automatically be used for cvmfs cache, and
it can also be used as scratch workspace for jobs.

Make sure that there is a unique file for each running copy of cvmfsexec.
Create the file with commands like this:
```
truncate -s 600G scratch.img
mkfs.ext4 -F -O ^has_journal scratch.img
```
Choose a count of the number of gigabytes you want.  The default cache
size is 4000 megabytes, and the recommendation is to reserve an
additional 1000 megabytes plus 20%, so make it at least 6 gigabytes
and add any additional space you want to use as scratch workspace.
It's a sparse file so there's little penalty for creating it bigger if
the space is never used.
Then start cvmfsexec with `-m`.  For example, to mount only the
cvmfs configuration repository and run a shell do
```
cvmfsexec -m scratch.img --
```
Then check out `/e2fs`.

NOTE: although this functions easily and well, fuse2fs performance is
not great so if a more performant option is available that is likely
to be preferred.

## Better cvmfsexec operation on newer kernels (mode 3)

A caveat on older kernels (for example RHEL7.7 and older) is that a
kill -9  of all the processes will not clean up the mounts, and they
have to be separately unmounted later with `umountrepo` or `fusermount -u`.
On kernels >= 4.18 (for example RHEL8) or >= 3.10.0-1127 (for example on
RHEL7.8) the operation changes to do fuse mounts only inside of
unprivileged user namespaces, which always completely cleans up mounts
even with kill -9.  This also normally uses a pid namespace to ensure
that all fuse processes are always cleaned up when the command exits
(the exception is when running under docker and kubernetes default
configurations that "mask" /proc).

$CVMFSMOUNT/$CVMFSUMOUNT still send a request to a parent process to
mount/umount but it's not the original process, it's an intermediate
process that has fakeroot access in the user namespace.

## mountrepo/umountrepo without cvmfsexec (mode 1)

When not using cvmfsexec, but with fusermount available use
`mountrepo repository.name` to mount a repository.  Note that the osg
configuration requires "config-osg.opensciencegrid.org" to be mounted
first, and the egi configuration requires "config-egi.egi.eu".

If you are using a container system, bind mount $PWD/dist/cvmfs into the
container as /cvmfs.

To unmount all repositories, use `umountrepo -a`, or to unmount an
individual repository use `umountrepo repository.name`.  Make sure that
all the processes do not get killed or the repositories will remain
mounted but inaccessible.

Cache considerations for this mode are the same as with the cvmfsexec
command.

## Debugging

Syslog messages from cvmfs go in the `log` subdirectory alongside
`dist`.  A separate log file is created for each repository.  cvmfs
debugging logs can also be enabled in the usual way, by setting
CVMFS_DEBUGLOG in the cvmfs configuration.

## Running from docker

Docker supports unprivileged user namespaces including unprivileged fuse
mounts on the kernels that support it, without using the `--privileged`
option or adding capabilities.  The following set of docker options is
sufficient:

```
--security-opt seccomp=unconfined --security-opt systempaths=unconfined --device=/dev/fuse
```

If you have no need for running any setuid executables in docker then
you can improve security further by adding:
```
--security-opt no-new-privileges
```
Singularity and apptainer always have the equivalent protection enabled for the
containers they run.

# singcvmfs command (mode 4)

When a privileged setuid installation of singularity >= 3.4 is
available, the `singcvmfs` command can be used to mount cvmfs
repositories inside a container.  With singularity >= 3.6 and
RHEL >= 7.8 or a kernel >= 4.18
with unprivileged user namespaces enabled
this can also be used with an
unprivileged non-setuid singularity or apptainer installation.
The command line interface is different than cvmfsexec because it is
designed for ease of use by end users on a laptop/desktop and as a
drop-in replacement for singularity when it executes containers.

Put cvmfs repositories to mount comma-separated in a
`SINGCVMFS_REPOSITORIES` environment variable.  If a configuration
repository is needed it will be automatically mounted.  Then you can use
singcvmfs exactly like singularity with one of its exec, instance, run,
or shell commands (note: it cannot read an image from cvmfs).  For example,
once you have [made a singcvmfs distribution](#making-the-cvmfs-distribution)
the following should work (replace `centos:7` with a container matching
the operating system you're running on):

```
$ export SINGCVMFS_REPOSITORIES="grid.cern.ch,atlas.cern.ch"
$ singcvmfs -s exec -cip docker://centos:7 bash
Singularity> ls /cvmfs
atlas.cern.ch  config-osg.opensciencegrid.org  grid.cern.ch
Singularity> ls /cvmfs/atlas.cern.ch
repo
Singularity> exit

# or using singularity instances:
$ export SINGCVMFS_REPOSITORIES="grid.cern.ch,atlas.cern.ch"
$ singcvmfs -s instance start docker://centos:7 myexampleinstance
$ singcvmfs -s run instance://myexampleinstance ls /cvmfs
atlas.cern.ch  config-osg.opensciencegrid.org  grid.cern.ch
$ singcvmfs -s run instance://myexampleinstance ls /cvmfs/atlas.cern.ch
repo
$ singcvmfs -s instance stop myexampleinstance
```

The first time you run the above it will take a long time as singularity
downloads the image from dockerhub and cvmfs fills its cache, but
running it again should be very fast.

Alternatively, to make it easier to execute repeatedly interactively
from the command line, you can put the singularity container path in a
`SINGCVMFS_IMAGE` environment variable and leave out the singularity
command.  The image cannot come from cvmfs, but it can come from docker,
shub, a local image file, or a local "sandbox" unpacked image directory.
Then the usage is `singcvmfs [command]` where the default command is
$SHELL.  For example:

```
$ export SINGCVMFS_REPOSITORIES="grid.cern.ch,atlas.cern.ch"
$ export SINGCVMFS_IMAGE="docker://centos:7"
$ singcvmfs ls /cvmfs
atlas.cern.ch  config-osg.opensciencegrid.org  grid.cern.ch
$ singcvmfs ls /cvmfs/atlas.cern.ch
repo
```

There are other optional environment variables that may be set.
Run `singcvmfs -h` for more details.

Caveat: singcvmfs works by bind-mounting all of the files from the cvmfs
distribution into the container, including the fuse3 libraries.  It
expects other base system libraries to be available inside the
container, so if the container uses a different base OS distribution it
will likely not work unless compatible libraries are in the container.

Cache considerations: by default singcvmfs starts a cache manager for all
the cvmfs repositories it mounts, which means that caches cannot be shared
between different invocations of singcvmfs on the same machine.
This tends to be more of a problem than with the cvmfsexec command
because it is more common to run many payload jobs on a machine with
singularity than it is to run many pilots.  You can
select a different cache directory for each invocation by setting
SINGCVMFS_CACHEDIR.  Alternatively it's possible to use the [cvmfs alien
cache](https://cvmfs.readthedocs.io/en/stable/cpt-configure.html#alien-cache)
feature to share the cache, but you would have to make sure that the
cache doesn't grow too big and that it gets cleaned up at some point.
Make sure that caches are not on shared filesystems because they're
likely to do too many metadata operations.

There is also a SINGCVMFS_CACHEIMAGE variable which can be set to an
ext3 filesystem image for the cache.  If set, it must point to a
file with such a filesystem including a directory that is writable by
the user or a `shared` directory within it that is writable by the user.
The directory within the filesystem can be changed by setting
SINGCVMFS_CACHEDIR but defaults to the root directory, `/`.
If running as an unprivileged user, this can directory can be created
using the `mkfs.ext3 -d` option.  Unfortunately the version of
`mkfs.ext3` on RHEL7 is too old, but it can be compiled from
[source](https://github.com/tytso/e2fsprogs/blob/master/INSTALL).
Then to make the image file these commands should work:

```
$ truncate -s 6G scratch.img
$ mkdir -p tmp/shared
$ mkfs.ext3 -F -O ^has_journal -d tmp scratch.img
```

By default the cvmfs logs are written to a top-level `log` directory, alongside
the top-level `dist` directory. The variable `SINGCVMFS_LOGDIR` can be used to
write them to a different directory, which will be created if it doesn't exist.
