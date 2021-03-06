# Count boot instructions

- <https://www.quora.com/How-many-instructions-does-a-typical-Linux-kernel-boot-take>
- <https://github.com/cirosantilli/chat/issues/31>
- <https://rwmj.wordpress.com/2016/03/17/tracing-qemu-guest-execution/>
- `qemu/docs/tracing.txt` and `qemu/docs/replay.txt`
- <https://stackoverflow.com/questions/39149446/how-to-use-qemus-simple-trace-backend/46497873#46497873>

Best attempt so far:

    time ./runqemu -n -e 'init=/poweroff.out' -- -trace exec_tb,file=trace && \
      time ./qemu/scripts/simpletrace.py qemu/trace-events trace >trace.txt && \
      wc -l trace.txt && \
      sed '/0x1000000/q' trace.txt >trace-boot.txt && \
      wc -l trace-boot.txt

Notes:

-   `-n` is a good idea to reduce the chances that you send unwanted non-deterministic mouse or keyboard clicks to the VM.

-   `-e 'init=/poweroff.out'` is crucial as it reduces the instruction count from 40 million to 20 million, so half of the instructions were due to userland programs instead of the boot sequence.

    Without it, the bulk of the time seems to be spent in setting up the network with `ifup` that gets called from `/etc/init.d/S40network` from the default Buildroot BusyBox setup.

    And it becomes even worse if you try to `-net none` as recommended in the 2.7 `replay.txt` docs, because then `ifup` waits for 15 seconds before giving up as per `/etc/network/interfaces` line `wait-delay 15`.

-   `0x1000000` is the address where QEMU puts the Linux kernel at with `-kernel` in x86.

    It can be found from:

        readelf -e buildroot/output.x86_64~/build/linux-*/vmlinux | grep Entry

    TODO confirm further. If I try to break there with:

        ./rungdb *0x1000000

    but I have no corresponding source line. Also note that this line is not actually the first line, since the kernel messages such as `early console in extract_kernel` have already shown on screen at that point. This does not break at all:

        ./rungdb extract_kernel

    It only appears once on every log I've seen so far, checked with `grep 0x1000000 trace.txt`

    Then when we count the instructions that run before the kernel entry point, there is only about 100k instructions, which is insignificant compared to the kernel boot itself.

-   We can also discount the instructions after `init` runs by using `readelf` to get the initial address of `init`. One easy way to do that now is to just run:

        ./rungdb-user kernel_module-1.0/user/poweroff.out main

    And get that from the traces, e.g. if the address is `4003a0`, then we search:

        grep -n 4003a0 trace.txt

    I have observed a single match for that instruction, so it must be the init, and there were only 20k instructions after it, so the impact is negligible.

This works because we have already done the following with QEMU:

-   `./configure --enable-trace-backends=simple`. This logs in a binary format to the trace file.

    It makes 3x execution faster than the default trace backend which logs human readable data to stdout.

    This also alters the actual execution, and reduces the instruction count by 10M TODO understand exactly why, possibly due to the `All QSes seen` thing.

-   the simple QEMU patch mentioned at: <https://rwmj.wordpress.com/2016/03/17/tracing-qemu-guest-execution/> of removing the `disable` from `exec_tb` in the `trace-events` template file in the QEMU source

Possible improvements:

-   to disable networking. Is replacing `init` enough?

    - <https://superuser.com/questions/181254/how-do-you-boot-linux-with-networking-disabled>
    - <https://superuser.com/questions/684005/how-does-one-permanently-disable-gnu-linux-networking/1255015#1255015>

    `CONFIG_NET=n` did not significantly reduce instruction, so maybe replacing `init` is enough.

-   logging with the default backend `log` greatly slows down the CPU, and in particular leads to this during kernel boot:

        All QSes seen, last rcu_sched kthread activity 5252 (4294901421-4294896169), jiffies_till_next_fqs=1, root ->qsmask 0x0
        swapper/0       R  running task        0     1      0 0x00000008
         ffff880007c03ef8 ffffffff8107aa5d ffff880007c16b40 ffffffff81a3b100
         ffff880007c03f60 ffffffff810a41d1 0000000000000000 0000000007c03f20
         fffffffffffffedc 0000000000000004 fffffffffffffedc ffffffff00000000
        Call Trace:
         <IRQ>  [<ffffffff8107aa5d>] sched_show_task+0xcd/0x130
         [<ffffffff810a41d1>] rcu_check_callbacks+0x871/0x880
         [<ffffffff810a799f>] update_process_times+0x2f/0x60

    in which the boot appears to hang for a considerable time.

-   Confirm that the kernel enters at `0x1000000`, or where it enters. Once we have this, we can exclude what comes before in the BIOS.
