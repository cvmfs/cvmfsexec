# fusermount-cvmfs

This package is for mounting cvmfs as an unprivileged user, on systems
where fusermount is available.

The cvmfs code and configuration is installed in a "dist" subdirectory
from where the scripts are.  The easiest way to create the dist
directory is to use "makedist".  It takes a parameter of "osg", "egi",
or "default" to install the latest cvmfs and configuration rpm from
one of those three sources.  Requires "rpm2cpio".

To customize any cvmfs configuration settings, put them in
dist/etc/cvmfs/default.local.  In particular you'll probably want
to set CVMFS_HTTP_PROXY, although the default is to use WLCG Web
Proxy Auto Discovery.

To mount a repository use "mountrepo repository.name".  Note that the
osg configuration requires "config-osg.opensciencegrid.org" to be
mounted first, and the egi configuration requires "config-egi.egi.eu".

If you're using a container system, bind mount $PWD/dist/cvmfs into
the container as /cvmfs.

To unmount all repositories, use "umountrepo -a", or to unmount an
individual repository use "umountrepo repository.name".

Enjoy!
