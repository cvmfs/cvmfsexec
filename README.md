# cvmfsexec

This package is for mounting cvmfs as an unprivileged user, on systems
where fusermount or unprivileged namespace fuse mounts are available.
The cvmfsexec command itself additionally requires unprivileged user
namespaces, but mountrepo and umountrepo also work separately with only
fusermount.  On newer kernels (more about that in the next section)
fusermount is not needed and cvmfsexec instead uses unprivileged
namespace fuse mounts.

The cvmfs code and configuration is installed in a "dist" subdirectory
under where the scripts are.  The easiest way to create the dist
directory is to use makedist.  It takes a parameter of "osg", "egi",
or "default" to install the latest cvmfs and configuration rpm from
one of those three sources.  Requires rpm2cpio.

To customize any cvmfs configuration settings, put them in
dist/etc/cvmfs/default.local.  In particular you may want to set
CVMFS_HTTP_PROXY, although the default is to use WLCG Web Proxy Auto
Discovery.  You may also want to set CVMFS_QUOTA_LIMIT which defaults
to 4000 MB.

To execute a command in an environment where cvmfs repositories are
mounted at "/cvmfs" and automatically unmounted upon exit, use
`cvmfsexec repository.name ... -- [command]` where the default command
is $SHELL.  It will automatically mount the configuration repository
if one is defined. 

Inside the command you can mount additional repositories by using
`$CVMFSMOUNT repository.name`.  Since the mounts have to happen outside
the user namespace, it actually sends a message to the original process
to mount, and makes the current process wait until completion.
Repositories that are already mounted are ignored.  You can also unmount
repositories from within the command with `$CVMFSUMOUNT repository.name`.

If you invoke additional processes within the original process that are
not trustworthy, such as a user payload that is invoked with singularity
--contain, then close the $CVMFSEXEC_CMDFD file descriptor for those
processes.  This can be done in bash with `{CVMFSEXEC_CMDFD}>&-`.

## Better operation on kernels >= 4.18

A caveat on older kernels (for example RHEL7) is that a kill -9  of
all the processes will not clean up the mounts, and they have to be
separately unmounted later with umountrepo or fusermount -u.  On
kernels >= 4.18 (for example RHEL8) the operation changes to do fuse
mounts only inside of unprivileged user namespaces, which always
completely cleans up mounts even with kill -9.

$CVMFSMOUNT/$CVMFSUMOUNT still send a request to a parent process to
mount/umount but it's not the original process, it's an intermediate
process that has fakeroot access in the user namespace.

## mountrepo/umountrepo without cvmfsexec

When not using cvmfsexec, use `mountrepo repository.name` to mount a
repository.  Note that the osg configuration requires
"config-osg.opensciencegrid.org" to be mounted first, and the egi
configuration requires "config-egi.egi.eu".

If you're not using cvmfsexec but are using a container system, bind
mount $PWD/dist/cvmfs into the container as /cvmfs.

To unmount all repositories, use `umountrepo -a`, or to unmount an
individual repository use `umountrepo repository.name`.
