# cvmfsexec

This package is for mounting cvmfs as an unprivileged user, on systems
where at least fusermount is available.  The cvmfsexec command itself
additionally requires unprivileged user namespaces, but mountrepo and
umountrepo also work separately with only fusermount.

The cvmfs code and configuration is installed in a "dist" subdirectory
under where the scripts are.  The easiest way to create the dist
directory is to use "makedist".  It takes a parameter of "osg", "egi",
or "default" to install the latest cvmfs and configuration rpm from
one of those three sources.  Requires "rpm2cpio".

To customize any cvmfs configuration settings, put them in
dist/etc/cvmfs/default.local.  In particular you may want to set
CVMFS_HTTP_PROXY, although the default is to use WLCG Web Proxy Auto
Discovery.  You may also want to set CVMFS_QUOTA_LIMIT which defaults
to 4000 MB.

To execute a command in an environment where cvmfs repositories are
mounted at "/cvmfs" and automatically unmounted upon exit, use
"cvmfsexec repository.name ... -- [command]" where the default command
is $SHELL.

Inside the command you can mount additional repositories by using
$CVMFSEXEC instead of just "cvmfsexec".  That variable does "exec" on a
complete path to cvmfsexec.  Since the mounts have to happen outside the
user namespace, it actually sends a message to the other process to
mount and execute the command, and makes the current process wait until
completion.  Each invocation that adds a repository to mount adds an
additional process in the process tree.

When not using cvmfsexec, use "mountrepo repository.name" to mount a
repository.  Note that the osg configuration requires
"config-osg.opensciencegrid.org" to be mounted first, and the egi
configuration requires "config-egi.egi.eu".

If you're not using cvmfsexec but are using a container system, bind
mount $PWD/dist/cvmfs into the container as /cvmfs.

To unmount all repositories, use "umountrepo -a", or to unmount an
individual repository use "umountrepo repository.name".
