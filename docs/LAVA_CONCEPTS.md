# How LAVA works — concepts, using this lab as the example

Everything in this document is grounded in the kernelci-1 lab: the Orange Pi RV2 bring-up ([ADDING_ORANGEPI_RV2.md](/docs/ADDING_ORANGEPI_RV2.md)) is the running example, and the last section maps every failed job from that bring-up to the concept it teaches. Read this once and the copy-paste in the other guides turns into things you could have written yourself.

## The one-sentence mental model

LAVA is an expect script with a database in front of it: everything it knows about a running board arrives as bytes on the serial console, everything it does to the board is either a shell command on the worker (power on/off, connect to serial) or a line typed into that console, and everything else — device types, jobs, results, the web UI — is bookkeeping around that loop.

Keep that in mind and both halves of LAVA make sense. The *server* half is Django bookkeeping. The *dispatcher* half is `pexpect`: send a line, wait for a pattern, repeat. Almost every problem you will ever debug is a pattern problem — the right bytes not arriving, or arriving in a shape the pattern doesn't match.

## The machines and daemons in this lab

Three machines participate in a job:

- **kernelci-1** runs both halves of LAVA. The *server* half is `lava-server-gunicorn` (the Django app behind Apache — the UI at kernelci.cloud-v.co and the XML-RPC/REST API), `lava-scheduler` (matches queued jobs to idle healthy devices), `lava-publisher` (event stream), and PostgreSQL. The *worker* half is `lava-worker` (the dispatcher), registered as `worker-2`. Server and worker only speak HTTP to each other, which is why they can be one box today and split later without changing anything else.
- **The board** (orangepi-rv2-1) has no agent, no daemon, nothing installed. LAVA reaches it exclusively through side channels: the Tuya smart-strip outlet for power, ser2net→telnet for serial, and TFTP/NFS served *from* the worker when the board's own U-Boot or kernel asks for files.
- **Your workstation** is outside LAVA entirely. `rv2-build-and-send-lava.sh` builds the kernel and kselftest tarball and scp's them into `/srv/lava/rv2/` on kernelci-1; jobs then reference them as `file:///srv/lava/rv2/...`. LAVA does not build anything — it consumes artifacts.

Supporting services on kernelci-1, all ordinary Debian daemons that LAVA merely *uses*: `ser2net` (serial → TCP), `tftpd-hpa` (serves `/srv/tftp`), `nfs-kernel-server` (exports `/var/lib/lava/dispatcher/tmp`, an export the LAVA package itself adds to `/etc/exports` — never add a second one).

## Device types, devices, workers

Three layers, from generic to specific:

- A **device type** (`orangepi-rv2`) is *how to boot this kind of board*. It lives as a Jinja2 template in `/etc/lava-server/dispatcher-config/device-types/` and almost never starts from scratch: ours is 40 lines of `{% set ... %}` on top of the packaged `base-uboot.jinja2` (in `/usr/share/lava-server/device-types/`), which contains the whole generic U-Boot conversation. Your variables are the parameters of that conversation: where to load things (`booti_kernel_addr`), how to fetch them (`uboot_tftp_commands` spelling out `tftpboot`, because `tftp` is ambiguous on SpacemiT's U-Boot), what the prompt looks like (`bootloader_prompt = '=>'`), how to cut power (`power_on_command` → `tuya-power`), what to put on the kernel command line (`base_kernel_args`, `base_nfsroot_args`, `base_ip_args`).
- A **device** (`orangepi-rv2-1`) is *one physical board*: its dictionary in `dispatcher-config/devices/` extends the type and would carry per-board differences (this ser2net port, that PDU outlet). With one RV2 in the lab those live in the type; a second RV2 is the moment they move down into the dictionaries.
- A **worker** (`worker-2`) is *which machine's cables the board hangs off*. Per-worker settings live in `/etc/lava-server/dispatcher.d/worker-2/dispatcher.yaml` — ours sets `dispatcher_ip: 192.168.2.2`, the address on the isolated board link, so `{SERVER_IP}` in rendered commands points where the board can actually reach.

The database rows (`lava-server manage device-types add`, `devices add`) just register names and wiring between these files. The rendered result — what the dispatcher actually receives when a job starts — is one flat YAML you can and should look at:

```
lavacli -i local devices dict get orangepi-rv2-1 --render
```

When a template change doesn't do what you expect, render first. The `=> setenv ...` lines in every job log are exactly this render with `{KERNEL_ADDR}`-style placeholders substituted; the `substitutions:` block printed at `bootloader-overlay` shows every placeholder and its value.

## The job, the scheduler, and health

A **job** is a YAML document: metadata (device type, timeouts, visibility) plus an `actions:` list that is almost always `deploy` → `boot` → `test`. Submission (`lavacli -i local jobs submit --follow job.yaml`) validates it against the device type — job 295 died here, in `validate`, before any hardware was touched — then queues it.

The **scheduler** assigns queued jobs to devices that are Idle *and healthy*. Health is a per-device state (GOOD / UNKNOWN / BAD / MAINTENANCE) maintained by the **health check**: the job YAML installed at `/etc/lava-server/dispatcher-config/health-checks/orangepi-rv2.yaml` (one per device *type*). Whenever a device is UNKNOWN — after `devices update --health UNKNOWN`, or after a job ends badly — the scheduler runs the health check before anything else; pass → GOOD and the queue flows, fail → BAD and user jobs sit in Submitted until you fix the lab. That is the whole gatekeeping mechanism: keep the health check minimal (ours is the TFTP-ramdisk boot plus smoke tests) so it tests the lab, not the kernel-of-the-day.

Job state (Submitted → Scheduled → Running → Complete/Incomplete) is about the *pipeline*; test results are about the *kernel*. Job 304 was `Complete` with failing test cases — correct: the machinery worked, the kernel has driver gaps. `Incomplete` means the machinery itself broke (jobs 296–298, 302).

## What `deploy` actually does

Everything in `deploy` happens on the worker; the board is still powered off. For our `to: tftp` deploy, in log order:

1. **Overlay build** (`lava-overlay`): LAVA creates `/lava-<jobid>/` — a directory of helper scripts (`lava-test-runner`, `lava-test-case`, ...) plus your test definitions, cloned now (`git-repo-action` fetching Linaro test-definitions) with the `params:` you set baked into a runner script. This is packed as a tarball. It is how "run kselftest" becomes something a serial console can do.
2. **Downloads**: each `url:` in the deploy (kernel, dtb, ramdisk, nfsrootfs, modules) is fetched into `/srv/tftp/<jobid>/tftp-deploy-*/`, decompressing what it will repack.
3. **prepare-tftp-overlay**: the `nfsrootfs` tarball is extracted to `/var/lib/lava/dispatcher/tmp/<jobid>/extract-nfsrootfs-*/` — this directory *is* the board's future root filesystem, served over NFS thanks to the package's `/etc/exports` entry. The `modules:` tarball is extracted on top of it (that is how `kselftest.tar.xz` becomes `/opt/kselftest` — the field is named for kernel modules but it is just "unpack this tarball into the rootfs"). The overlay tarball lands in it too (`/lava-304`). The same happens to the ramdisk when the job is ramdisk-rooted.
4. **Ramdisk repack**: the initrd is rebuilt with the overlay inside where applicable, then wrapped in a U-Boot legacy header — the `mkimage -A riscv -T ramdisk` line. This is why the template sets `uboot_mkimage_arch: riscv` and why `booti` takes the ramdisk address with no size.

Nothing is permanent: `/srv/tftp/<jobid>` and `/var/lib/lava/dispatcher/tmp/<jobid>` are deleted at job end, which is why every job re-extracts the 138 MB rootfs (9 s here — fine).

## What `boot` actually does

Now the expect script runs, against the serial console:

1. **connect-device**: runs the type's `connection_commands` — literally `telnet localhost 5002`, a shell command whose stdin/stdout become the pexpect channel. ser2net bridges that TCP port to `/dev/serial/by-id/usb-1a86...`. The accepter must be `telnet(rfc2217),...` — raw `tcp` delivered `\r\n` line endings that doubled every prompt, and LAVA "saw" two prompts per command and ran ahead (job 298).
2. **reset-device**: runs `hard_reset_command` — the Tuya off/sleep/on. LAVA has no idea whether it worked; it only knows what the serial port says next.
3. **bootloader-interrupt**: waits for `interrupt_prompt`. Our board needs no interrupting — the saved `bootcmd='echo LAVA'` drops it at `=>` — so the template sets `uboot_needs_interrupt: False` and LAVA just waits for `=>`.
4. **bootloader-commands**: types the rendered command list one line at a time, waiting for `=>` after each, with a long list of error patterns (`TFTP error`, `Bad Linux RISCV Image magic!`, ...) that would fail the job immediately. The last command (`booti ...`) switches the expected pattern to `Starting kernel`.
5. **auto-login-action / login-action**: waits for `Linux version [0-9]` (kernel is alive), then parses the console for kernel panic/oops patterns while waiting for `auto_login.login_prompt` (`login:`), sends the username, and finally waits for one of the job's `prompts:`. The prompt list is a contract with the *rootfs*: the buildroot ramdisk gives `~ #`, the Debian trixie-kselftest image auto-logs root into a busybox-style `/ # ` — matching bash's `root@host:~#` instead timed the job out after a fully successful boot (job 302).

The kernel command line the board received was assembled by the base template from your variables: `base_kernel_args` + method-specific pieces (`root=/dev/nfs`, `base_nfsroot_args` with the extracted-rootfs path substituted in, `base_ip_args`). One subtlety that cost us three minutes per boot until fixed: `ip=...eth0` is consumed twice — by the kernel (early, before udev) and by the Debian initrd's scripts (after udev may have renamed eth0 → end0). `net.ifnames=0` in `base_kernel_args` keeps the name stable for both.

## What `test` actually does

Still the same serial channel — no ssh, no agent. `lava-test-shell` sets a private prompt, sources `/lava-<jobid>/environment`, and runs `lava-test-runner`, which executes each test definition's `run: steps:` as a shell script on the board. Results travel *back over the console* as magic strings:

```
<LAVA_SIGNAL_STARTRUN 0_kselftest 304_1.1.4.1>
<LAVA_SIGNAL_TESTCASE TEST_CASE_ID=size_get_size RESULT=pass>
```

The dispatcher greps these out of the byte stream and posts them to the server as test cases — that is the entire "results integration". Anything a script reports with `lava-test-case NAME --result pass|fail` (or, as with kselftest, a parser that converts KTAP output into these signals via `send-to-lava.sh`) becomes a queryable result. This is also why console discipline matters end to end: a rootfs that spews log noise into the middle of a signal line can corrupt results.

A test definition (like Linaro's `automated/linux/kselftest/kselftest.yaml`) is nothing deeper than: metadata, `params:` with defaults (overridable from the job), and steps that run a shell script from the cloned repo. When one misbehaves — the `SKIPFILE: ""` bug that ran `wget ''`, or `TST_CMDFILES` needing space separation because the script does `for test in ${TST_CMDFILES}` — you debug it like any shell script, by reading it in the test-definitions repo.

## lavacli and the API

`lavacli` is a thin client for the server's XML-RPC API at `/RPC2/`. Identities (`lavacli identities add`) live per-user in `~/.config/lavacli.yaml` — which is why `sudo lavacli` sees none of them. The subcommands used daily here: `jobs submit --follow`, `jobs logs <id>`, `jobs show <id>`, `devices dict get --render`, and on the server side (management, not API) `lava-server manage devices update --health UNKNOWN`.

## Every failure of the RV2 bring-up, as a lesson

| Job | Symptom | Concept it teaches |
|---|---|---|
| 295 | `validate` error: "No booti parameters available", `commands: tftp` → TypeError | Jobs are validated against the *rendered device type* before hardware is touched. The boot method is named `ramdisk`/`nfs` (not `tftp`), and it only exists if the template supplies its variables (`booti_*_addr`). |
| 296 | ser2net "Device open failure" on port 5001 | `connection_commands` is an arbitrary shell command; LAVA knows nothing about serial ports. The template said 5001, ser2net listened on 5002. |
| 297 | Power-cycled, then 293 s of silence at `bootloader-interrupt` | `power_*_command`s are opaque to LAVA — exit 0 means "the command ran", not "the board rebooted". The RV2 was on outlet 2, not 1. Only the serial stream tells the truth. |
| 298 | U-Boot got `setenv bootargs`/`booti` typed into it mid-`tftpboot` | pexpect: the raw-`tcp` ser2net accepter delivered `\r\n`, every prompt matched twice, and LAVA believed each command had finished one prompt early. `telnet(rfc2217)` fixed the framing. |
| 299 | **green** (ramdisk boot + smoke) | The health check baseline. |
| 302 | Full boot + NFS mount + auto-login, then `login-action timed out` | `prompts:` is a contract with the rootfs: this image gives `/ # `, not `root@host:~#`. Also: the eth0→end0 udev rename stalled the initrd ~3 min → `net.ifnames=0`. |
| 303 | kselftest died in a second: `wget ''` / "Failed to fetch" | Test definitions are just shell scripts; this one mishandles an empty `-S`. Params are your interface to them: `SKIPFILE: "none"`. |
| 304 | Complete; `shardfile-exec: fail`, 7 `dt` node fails | Deploy ships only what you built (`exec` never cross-compiled into the tarball), and a failing *test case* on a `Complete` job is signal about the kernel, not the lab — the 7 unprobed DT nodes are the RV2's mainline driver-gap list. |

## Debugging checklist

When a job goes wrong, find the first line of the log that deviates and ask which layer owns it: validation error → job YAML vs rendered template (`devices dict get --render`); nothing on serial after `pdu-reboot` → power path or console path (open `telnet localhost 5002` yourself and toggle the outlet); U-Boot misbehaving → replay the exact `=>` commands from the log by hand over that same telnet; kernel boots but LAVA times out → your `prompts:`/`auto_login` vs what the rootfs actually prints (the log shows the real bytes); test weirdness → read the test definition's script and its params. The job log is a verbatim transcript of the only channel LAVA has — everything you need is in it.
