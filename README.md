# cvmfsexec package

This package is for mounting cvmfs as an unprivileged user, without the
cvmfs package being installed by a system administrator.  The package can
do this in 4 different ways:

1. On systems where only fusermount is available, the `mountrepo` and
   `umountrepo` commands can be used to mount cvmfs repositories in the
   user's own file space.  That path can then be bindmounted at /cvmfs
   by a container manager such as singularity.
2. On systems where fusermount is available and unprivileged user
   namespaces are enabled, but unprivileged namespace fuse mounts are not
   available (in particular RHEL <=7.7 with
   `sysctl user.max_user_namespaces` > 0),
   the `cvmfsexec` command can mount cvmfs repositories, map them into
   /cvmfs, and unmount them when it exits.  singularity may even be
   run unprivileged from cvmfs from within cvmfsexec (it has to run
   unprivileged because setuid-root does not work inside a user
   namespace).  The main disadvantage is
   that if the processes are hard-killed (kill -9), mountpoints are left
   behind and are difficult to clean up.
3. On systems where unprivileged namespace fuse mounts are available
   (newer kernels >= 4.18 as on RHEL8 or >= 3.10.0-1127 as on RHEL 7.8),
   the `cvmfsexec` command can entirely manage mounting and unmounting
   of cvmfs repositories in the namespace, so if they get killed
   everything gets cleanly unmounted.  fusermount is not needed in this
   case.
4. On systems that have no fusermount nor unprivileged user namespace
   fuse mounts but do have an setuid installation of singularity >= 3.4,
   an entirely separate command in this package `singcvmfs` can mount
   cvmfs repositories inside a container using the `singularity
   --fusemount` feature.  With singularity >= 3.6 and RHEL >= 7.8 this
   can also be used with unprivileged singularity.

# Making the cvmfs distribution

All of the ways this package supports unprivileged cvmfs make use of a
copy of the cvmfs software.  The cvmfs software and configuration are
expected to be in a `dist` subdirectory under where the scripts are.  The
easiest way to create the dist directory is to use `makedist`.  It takes a
parameter of `osg`, `egi`, or `default` to download the latest cvmfs and
configuration rpm from one of those three sources.  Requires rpm2cpio.

By default a distribution for `cvmfsexec` and `mountrepo/umountrepo` is
created.  To instead make a distribution for `singcvmfs`, add the `-s`
makedist option.

To customize any cvmfs configuration settings, put them in
`dist/etc/cvmfs/default.local`.  In particular you may want to set
CVMFS_HTTP_PROXY, although the default is to use WLCG Web Proxy Auto
Discovery.  You may also want to set CVMFS_QUOTA_LIMIT, otherwise the
default is 4000 MB.  The default CVMFS_CACHE_BASE for the cache
shared between the cvmfs repository is in under the dist directory,
`dist/var/lib/cvmfs`.  Make sure that the cache does not get shared
between multiple machines.

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

# cvmsexec command

The cvmfsexec command requires unprivileged user namespaces.  On RHEL8
unprivileged user namepaces (and user namespace fuse mounts) are
available by default, but on RHEL7 they need to be enabled by setting a
sysctl parameter as detailed in the 
[OSG unprivileged singularity instructions](https://opensciencegrid.org/docs/worker-node/install-singularity/#enabling-unprivileged-singularity).
In addition cvmfsexec requires fusermount on kernels older than
those that come with RHEL7.8.

To execute a command in an environment where cvmfs repositories are
mounted at /cvmfs and automatically unmounted upon exit, use
`cvmfsexec repository.name ... -- [command]` where the default command
is $SHELL.  It will automatically mount the configuration repository if
one is defined.  

Inside the command you can mount additional repositories by using
`$CVMFSMOUNT repository.name`.  Since the mounts have to happen outside
the user namespace, it actually sends a message to the original process
to mount, and makes the current process wait until completion.
Repositories that are already mounted are ignored.  You can also unmount
repositories from within the command with `$CVMFSUMOUNT repository.name`.

If you invoke additional processes within the original process that are
not trustworthy, such as user payloads that are invoked with
`singularity --contain`, then close the $CVMFSEXEC_CMDFD file descriptor
for those processes.  This can be done in bash with
`exec {CVMFSEXEC_CMDFD}>&-`.

Note that setuid-root programs do not work inside an unprivileged user
namepace, so if you use singularity it has to be run unprivileged.

## Better cvmfsexec operation on newer kernels

A caveat on older kernels (for example RHEL7.7 and older) is that a
kill -9  of all the processes will not clean up the mounts, and they
have to be separately unmounted later with `umountrepo` or `fusermount -u`.
On kernels >= 4.18 (for example RHEL8) or >= 3.10.0-1127 (for example on
RHEL7.8) the operation changes to do fuse mounts only inside of
unprivileged user namespaces, which always completely cleans up mounts
even with kill -9.  This also uses a pid namespace to ensure that all
fuse processes are always cleaned up when the command exits.

$CVMFSMOUNT/$CVMFSUMOUNT still send a request to a parent process to
mount/umount but it's not the original process, it's an intermediate
process that has fakeroot access in the user namespace.

## mountrepo/umountrepo without cvmfsexec

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

# singcvmfs command

When a privileged setuid installation of singularity >= 3.4 is
available, the `singcvmfs` command can be used to mount cvmfs
repositories inside a container.  With singularity >= 3.6 and
RHEL >= 7.8 or a kernel >= 4.18 this can also be used with an
unprivileged non-setuid singularity installation.
The command line interface is different than cvmfsexec because it is
designed for ease of use by end users on a laptop/desktop and as a
drop-in replacement for singularity when it executes containers.

Put cvmfs repositories to mount comma-separated in a
`SINGCVMFS_REPOSITORIES` environment variable.  If a configuration
repository is needed it will be automatically mounted.  Then you can use
singcvmfs exactly like singularity with one of its exec, run, or shell
commands (note: it cannot read an image from cvmfs).  For example:

```
$ export SINGCVMFS_REPOSITORIES="grid.cern.ch,atlas.cern.ch"
$ singcvmfs -s exec -cip docker://centos:7 ls /cvmfs
atlas.cern.ch  config-osg.opensciencegrid.org  grid.cern.ch
$ singcvmfs ls /cvmfs/atlas.cern.ch
repo
```
The first time you run the above it will take a long time as singularity
downloads the image from dockerhub and cvmfs mounts the repositories,
but running it again should be very fast.

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
