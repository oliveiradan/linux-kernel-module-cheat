# gdbserver

Step debug userland processes to understand how they are talking to the kernel.

In guest:

    /gdbserver.sh /myinsmod.out /hello.ko

In host:

    ./rungdbserver kernel_module-1.0/user/myinsmod.out

You can find the executable with:

    find buildroot/output.x86_64~/build -name myinsmod.out

TODO: automate the path finding:

-   using the executable from under `buildroot/output.x86_64~/target` would be easier as the path is the same as in guest, but unfortunately those executables are stripped to make the guest smaller. `BR2_STRIP_none=y` should disable stripping, but make the image way larger.

-   `outputx86_64~/staging/` would be even better than `target/` as the docs say that this directory contains binaries before they were stripped. However, only a few binaries are pre-installed there by default, and it seems to be a manual per package thing.

    E.g. `pciutils` has for `lspci`:

        define PCIUTILS_INSTALL_STAGING_CMDS
            $(TARGET_MAKE_ENV) $(MAKE1) -C $(@D) $(PCIUTILS_MAKE_OPTS) \
                PREFIX=$(STAGING_DIR)/usr SBINDIR=$(STAGING_DIR)/usr/bin \
                install install-lib install-pcilib
        endef

    and the docs describe the `*_INSTALL_STAGING` per package config, which is normally set for shared library packages.

    Feature request: <https://bugs.busybox.net/show_bug.cgi?id=10386>

An implementation overview can be found at: <https://reverseengineering.stackexchange.com/questions/8829/cross-debugging-for-mips-elf-with-qemu-toolchain/16214#16214>

## gdbserver different archs

As usual, different archs work with:

    ./rungdbserver -a arm kernel_module-1.0/user/myinsmod.out

## gdbserver BusyBox

BusyBox executables are all symlinks, so if you do on guest:

    /gdbserver.sh ls

on host you need:

    ./rungdbserver busybox-1.26.2/busybox

## gdbserver shared libraries

Our setup gives you the rare opportunity to step debug libc and other system libraries e.g. with:

    b open
    c

Or simply by stepping into calls:

    s

This is made possible by the GDB command:

    set sysroot ${buildroot_out_dir}/staging

which automatically finds unstripped shared libraries on the host for us.

See also: <https://stackoverflow.com/questions/8611194/debugging-shared-libraries-with-gdbserver/45252113#45252113>

## Debug userland process without gdbserver

QEMU `-gdb` GDB breakpoints are set on virtual addresses, so you can in theory debug userland processes as well.

- <https://stackoverflow.com/questions/26271901/is-it-possible-to-use-gdb-and-qemu-to-debug-linux-user-space-programs-and-kernel>
- <https://stackoverflow.com/questions/16273614/debug-init-on-qemu-using-gdb>

The only use case I can see for this is to debug the init process (and have fun), otherwise, why wouldn't you just use `gdbserver`? Known of direct userland debugging:

-   the kernel might switch context to another process, and you would enter "garbage"

-   TODO step into shared libraries. If I attempt to load them explicitly:

        (gdb) sharedlibrary ../../staging/lib/libc.so.0
        No loaded shared libraries match the pattern `../../staging/lib/libc.so.0'.

    since GDB does not know that libc is loaded.

Custom init process:

-   Shell 1:

        ./runqemu -d -e 'init=/sleep_forever.out' -n

-   Shell 2:

        ./rungdb-user kernel_module-1.0/user/sleep_forever.out main

BusyBox custom init process:

-   Shell 1:

        ./runqemu -d -e 'init=/bin/ls' -n

-   Shell 2:

        ./rungdb-user -h busybox-1.26.2/busybox ls_main

This follows BusyBox' convention of calling the main for each executable as `<exec>_main` since the `busybox` executable has many "mains".

BusyBox default init process:

-   Shell 1:

        ./runqemu -d -n

-   Shell 2:

        ./rungdb-user -h busybox-1.26.2/busybox init_main

This cannot be debugged in another way without modifying the source, or `/sbin/init` exits early with:

    "must be run as PID 1"

Non-init process:

-   Shell 1

        ./runqemu -d -n

-   Shell 2

        ./rungdb-user kernel_module-1.0/user/sleep_forever.out
        Ctrl + C
        b main
        continue

-   Shell 1

        /sleep_forever.out

This is of least reliable setup as there might be other processes that use the given virtual address.
