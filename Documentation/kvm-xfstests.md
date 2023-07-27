# kvm-xfstests: Running XFStests using virtualization

Please read the [kvm-quickstart](kvm-quickstart.md) instructions
first, since this will allow you to get started quickly.

If you don't have any familiarity with xfstests, you may also want to
read this [introduction to xfstests](what-is-xfstests.md).

## Installation

The kvm-xfstests system consists of a series of shell scripts, and a
test appliance virtual machine image.  You can build an image using
the build infrastructure in the xfstests-bld git repository, but if
you are just getting started, it will be much simpler if you let
kvm-xfstests automatically download a pre-compiled VM test appliance
image from [kernel.org](https://www.kernel.org/pub/linux/kernel/people/tytso/kvm-xfstests).

There are prebuilt test appliances for 32-bit and 64-bit x86 systems
(root_fs.img.i386 and root_fs.img.amd64) as well as for the 64-bit ARM
platform.  Thet test appliance images are installed in the
test-appliance directory, but it's easist to just allow kvm-xfstests
to download the image to the correct place, or to let the
build-appliance write the newly created test appliance image where it
should be located.  If you want to build your own test appliance VM,
see [building-rootfs.md](building-rootfs.md).

## Setup and configuration

The configuration file for kvm-xfstests is located
~/.config/kvm-xfstests.  A sample of the parameters that can be set in
the config file can be found in run-tests/config.kvm and
run-tests/config.common, which show the default values if they are not
overridden by settings in ~/.config/kvm-xfstests.

Perhaps the most important configuration variable to set is KERNEL.
This should point at the default location for the kernel that qemu
will boot to run the test appliance.  This is, in general, should be
the primary build tree that you use for kernel development.  If
kvm-xfstests is run from the top-level of a kernel build or source
tree where there is a built kernel, kvm-xfstests will use it.
Otherwise, it will use the kernel specified by the KERNEL variable.

To build a correctly configured kernel for use with kvm-xfstests, run
the commands:

        install-kconfig
        kbuild

If you wish to build kernels for the i386 or arm64 platforms, add
"--arch i386" or "--arch amd64" to install-kconfig and kbuild
commands.

By default, the scratch disks used by test-appliance will be set up
automatically, and are stored in the run-fstests directory with the
names vdb, vdc, vdd, ... up to vdg.  However, it is slightly faster to
use logical volumes.  To do this override the VDB..VDG variables in
the ~/.config/kvm-xfstests file.

        VG=closure

        VDB=/dev/$VG/test-4k
        VDC=/dev/$VG/scratch
        VDD=/dev/$VG/test-1k
        VDE=/dev/$VG/scratch2
        VDF=/dev/$VG/scratch3
        VDG=/dev/$VG/results

If you chose to do this, the logical volumes for VDB, VDC, VDD, and
VDG should be 5 gigabytes, while VDE and VDF should be 20 gigabyte
logical volumes.  The devices VDB and VDG should have an ext4 file
system created using the mkfs.ext4 command before you try running kvm-xfstests.

## Running kvm-xfstests

The kvm-xfstests shell script is in the run-fstests directory, and it
is designed to be run with the current working directory to be in the
run-fstests directory.  For convenience's sake, the Makefile in the
top-level directory of xfstests-bld will create a kvm-xfstests
shell script which can be copied into a convenient directory in your
PATH.  This shell script will set the KVM_XFSTESTS_DIR environment
variable so the auxiliary files can be found and then runs the
run-fstests/kvm-xfstests shell script.

Please run "kvm-xfstests help" to get a quick summary of the available
command-line syntax.  Not all of the available command-line options
are documented; some of the more specialized options will require that
you Read The Fine Source --- in particular, in the auxiliary script
file found in run-fstests/util/parse_cli.

### Running file system tests

The general form of the kvm-xfstests command to run tests in the test
appliance is:

        kvm-xfstests [-c <cfg>] [-g <group>]|[<tests>] ...


By default <cfg> defaults to all, which for ext4 will run the
following configurations: "4k", "1k", "ext3", "nojournal", "ext3conv",
"dioread_nolock, "data_journal", "inline", "bigalloc_4k", and
"bigalloc_1k".  You may specify a single configuration or a comma
separated list if you want to run a subset of all possible file system
configurations.

Tests can be specified using an xfstests group via "-g <group>", or
via one or more specific xfstests subtests (e.g., "generic/068").  The
most common test groups you will use are "auto" which runs all of the
tests that are suitable for use in an automated test run, and "quick"
which runs a subset of the tests designed for a fast smoke test.

For developer convenience, "kvm-xfstests smoke" is short-hand for
"kvm-xfstests -c 4k -g quick", which runs the fast subset of tests
using just 4k block file system configuration.  In addition
"kvm-xfstests full" is short-hand for "kvm-xfstests -g auto" which
runs all of the tests using a large set of file system configurations.
This will take quite a while, so it's best run overnight.  (Or it may
be better to run the full set of tests using gce-xfstests.)

### Running an interactive shell

The command "kvm-xfstests shell" will allow you to examine the tests
environment or to run tests manually, by booting the test kernel and
requesting that the test appliance VM start an interactive shell.

Any changes to the root partition will be reverted when you exit the
VM.  If you would like to modify the root_fs.img appliance
permanently, you can run "kvm-xfstests maint" instead.

You can run tests manually by looking at the environment variables set
in the /root/test-env file (which is sourced automatically when you
start an interactive shell).  You can then set FSTESTCFG and FSTESTSET
to control which tests you would like to run, and then run the test
runner script, /root/runtests.sh.  For example:

        % kvm-xfstests shell
        # FSTESTCFG="4k encrypt"
        # FSTESTSET="generic/001 generic/002 ext4/001"
        # /root/runtests.sh
        ...

To stop the VM, you can run the "poweroff" command, but a much faster way
to shut down the VM is to use the command sequence "C-a x" (that is,
Control-a followed by the character 'x'). 

## Local debugging ports

While kvm-xfstests is running, you can telnet to a number of TCP ports
(which are bound to localhost).  Ports 7500, 7501, and 7502 will
connect you to a shell prompts while the tests are running (if you
want to check on /proc/slabinfo, enable tracing, etc.)  You can also
use these ports in conjunction with "kvm-xfstests shell" if you want
additional windows to capture traces using ftrace.

You can also access the qemu monitor on port 7498, and you can debug the
kernel using remote gdb on localhost port 7499.  Just run "gdb
/path/to/vmlinux", and then use the command "target remote
localhost:7499".

Pro tips for using remote gdb: it's helpful to temporarily add
"EXTRA_CFLAGS += -O0" to fs/{ext4,jbd2}/Makefile, and use a kernel
config with debug features enabled via "kvm-xfstests install-kconfig
--debug".  In addition, you may need to add to your $HOME/.gdbinit the
line "add-auto-load-safe-path /path/to", where /path/to is the
directory containing the compiled vmlinux executable.  See
[Documentation/dev-tools/gdb-kernel-debugging.rst](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html)
in the kernel sources for more information.

## Log files

By default, when test results are saved in the run-fstests directory
with the filename log.<DATECODE>.

The get-results command will summarize the output from the log file.
It takes as an argument the name of the log file; if no log file is
specified, then the get-results command will display a summary of the
most recent log file.
