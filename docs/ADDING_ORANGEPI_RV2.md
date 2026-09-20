# Adding Orange Pi RV2 to LAVA (kernelci-1)

Complete, self-contained procedure. Run it top to bottom; every step ends with a check.

New to LAVA's moving parts? Read [LAVA_CONCEPTS.md](/docs/LAVA_CONCEPTS.md) first — it explains what each file below is *for*, using this exact bring-up as the running example.

The files this guide tells you to create are versioned in this repo, and the host-side steps (3 through 8, minus the one-time board work in step 3) are automated by [`ansible/kernelci-host.yml`](/ansible/kernelci-host.yml):

| In this repo | Installed as |
|---|---|
| `device_templates/orangepi-rv2.jinja2` | `/etc/lava-server/dispatcher-config/device-types/orangepi-rv2.jinja2` |
| `device_templates/orangepi-rv2-1.jinja2` | `/etc/lava-server/dispatcher-config/devices/orangepi-rv2-1.jinja2` |
| `Lava_job_template/orangepi-rv2-healthcheck.yaml` | `/etc/lava-server/dispatcher-config/health-checks/orangepi-rv2.yaml` (also submit it by hand to test) |
| `Lava_job_template/orangepi-rv2-kselftest.yaml` | `~/orangepi-rv2-kselftest.yaml` on the host (submitted with lavacli) |

So the fast path for a rebuilt host — or the next board, after copying and adjusting these four files — is: do step 1 (PDU script), step 3 (board U-Boot env, one-time), run the playbook, then jump to step 6 (manual netboot test) and step 8 (health check). The manual steps remain below because they are the debuggable version: when something breaks, you want to know which single command exercises the broken piece.

## What the setup looks like

The Orange Pi RV2 (Ky X1 = SpacemiT K1) boots the stock SpacemiT U-Boot (`U-Boot 2022.10ky`) from its on-board 16 MB QSPI NOR. That U-Boot already has the K1 Ethernet driver plus `dhcp` and `tftpboot`, so nothing is flashed to the board and no SD/USB media is inserted: LAVA power-cycles it, waits for the U-Boot prompt, TFTPs a kernel + DTB + ramdisk from the dispatcher and boots them.

Two decisions keep the LAVA side identical in shape to the existing `banana-pi-f3.jinja2`:

- The board's saved U-Boot env gets `bootcmd='echo LAVA'`, so after power-on it drops straight to `=>`. LAVA never has to interrupt an autoboot countdown (the stock build has a 0-second, keyed countdown that cannot be interrupted reliably).
- The device-type template feeds `base-uboot.jinja2` the pieces it expects (static `ipaddr` command, 64-bit `fdt_high`/`initrd_high`, `booti_*` load addresses — kernel `0x11000000`, ramdisk `0x21000000`, DTB `0x31000000`, the BPI-F3 layout — `tftpboot` load lines, `uboot_mkimage_arch = riscv`). The base template then composes the `ramdisk` and `nfs` methods itself and ends them with `{BOOTX}` = `booti <kernel> <ramdisk> <dtb>`, so the job just says `commands: ramdisk`. LAVA wraps the ramdisk in a U-Boot legacy header (`mkimage -A riscv -T ramdisk`) before serving it, which is why `booti` takes the ramdisk address without a size and why `uboot_mkimage_arch` must be `riscv`. Two base-template facts drove this: it names the boot method `ramdisk` (there is no `tftp` method), and its default load lines say `tftp`, which is ambiguous on this U-Boot because `tftpput`/`tftpsrv` are built in.

Verified on 2026-09-18: mainline 7.3.0-rc1 + upstream DTB + the KernelCI buildroot ramdisk boot to a shell on the RV2 with exactly this command list (PCIe root ports up, NVMe enumerated); the three kernel warnings seen (`Malformed early option 'earlycon'`, `mmc0: Failed to initialize a non-removable card`, `OPP table can't be empty`) are benign.

This coexists with the existing USB-Ubuntu development flow (`rv2-build-and-send.sh` / `rv2-install-on-board.sh` in `linux-kernel-notes`): the USB stick can stay in the board. With `bootcmd='echo LAVA'` it is simply no longer auto-booted; `run autoboot` at the `=>` prompt boots it on demand (see step 3). The two U-Boot lessons captured in `rv2-install-on-board.sh` — `fdt_high`/`initrd_high` must be all-Fs or the FDT relocation fails, and `earlycon=sbi console=ttyS0,115200` is needed for any output — are baked into the template's command list.

Values used below — replace where your setup differs:

| Item | Value |
|---|---|
| LAVA host (server + worker) | `kernelci-1`; office LAN `192.168.101.204` on `enx2c691d4f7cb5`, board link `192.168.2.2` on `enx00aa3458890a` |
| Board link (isolated, no DHCP) | `192.168.2.0/24` on the host's second USB NIC; RV2 static `192.168.2.6` (BPI-F3 scheme: host `.2`, F3 `.5`) |
| LAVA worker name | `worker-2` (`sudo lava-server manage workers list`) |
| Tuya strip outlet feeding the RV2 | `2` (verified 2026-09-20; re-check with step 1 if the strip is re-plugged) |
| RV2 USB-serial adapter | `/dev/serial/by-id/usb-1a86_USB_Serial-if00-port0` (a CH340; re-check with `ls -l /dev/serial/by-id/` if it changes) |
| ser2net port for the RV2 | `5002` (`5000`/`5001` were already taken by older entries) |
| lavacli identity | `IDENTITY` |
| Device type / device name | `orangepi-rv2` / `orangepi-rv2-1` |

Board wiring: 3-pin debug UART (GND/RX/TX, 3.3 V, 115200 8N1) to the USB-serial adapter, Ethernet to the board link (the host's second USB NIC `enx00aa3458890a`, directly or through the lab switch — not the office LAN), the board's 5 V/5 A USB-C adapter plugged into the Tuya strip outlet.

## 1. Power control (Tuya smart power strip)

Save the `tuya.py` script received from IT as `~/tuya.py` on the LAVA host. It needs two edits: the interpreter must point at a system-wide venv (the LAVA dispatcher runs as root and calls it), and a failed switch command must exit non-zero, otherwise LAVA carries on booting a board that never power-cycled.

```
sudo apt install -y python3-venv
sudo python3 -m venv /opt/tuya-env
sudo /opt/tuya-env/bin/pip install tuya-connector-python
sed -i '1s|.*|#!/opt/tuya-env/bin/python|' ~/tuya.py
sed -i 's|^\(\s*\)print("ERROR: Command failed")$|\1print("ERROR: Command failed"); sys.exit(1)|' ~/tuya.py
sudo install -m 0700 -o root -g root ~/tuya.py /usr/local/bin/tuya-power
sudo tuya-power status
```

Mode `0700` because the Tuya cloud secret is embedded in the file. `status` must list the five switches.

Find the RV2's outlet with the serial console open (`sudo minicom -D /dev/ttyUSB0 -b 115200`): the board must visibly reboot.

```
sudo tuya-power off 2; sleep 5; sudo tuya-power on 2
```

(Try the other numbers if that one doesn't reboot it.) Note the outlet number; it goes into the template in step 7. A board that is *not* reset prints nothing on the console and LAVA then times out waiting for `=>` — that was job 297.

Two Tuya facts worth knowing: the host needs outbound HTTPS to `openapi.tuyaeu.com` (ufw allows outgoing by default), and Tuya's IoT Core cloud subscription is a trial that expires periodically — when it lapses every call fails with a "subscription expired" error and every job on that strip fails at power-on. It is free to extend from the Tuya IoT platform.

## 2. Serial console (ser2net)

Use the adapter's stable path so a second USB-serial device can never steal `/dev/ttyUSB0`:

```
ls -l /dev/serial/by-id/
```

Add this connection to `/etc/ser2net.yaml` (the `*banner` reference is the `define: &banner` already in the file), with the RV2 adapter's by-id path and a TCP port no other entry in the file uses (`grep accepter /etc/ser2net.yaml`). The accepter must be `telnet(rfc2217),tcp,…`, not a raw `tcp,…` socket: with a raw socket LAVA's telnet client delivers `\r\n` to U-Boot, which runs an empty command and prints a spare prompt after every line, and LAVA's next "wait for prompt" is satisfied by that leftover — its commands then run ahead of the board (job 298: the `setenv bootargs`/`booti` lines were typed while `tftpboot` was still running and got discarded).

```yaml
connection: &orangepi-rv2-1
    accepter: telnet(rfc2217),tcp,localhost,5002
    enable: on
    options:
      banner: *banner
      kickolduser: true
      telnet-brk-on-sync: true
    connector: serialdev,
               /dev/serial/by-id/usb-1a86_USB_Serial-if00-port0,
               115200n81,local
```

```
sudo systemctl restart ser2net
telnet localhost 5002
```

Power-cycle the board with `tuya-power` while the telnet session is open: U-Boot output must appear. At `=>` press Enter once — exactly one new `=>` must appear (two means the accepter is still raw `tcp`). Quit telnet (`Ctrl-]`, `quit`) before using minicom again — only one of them can hold the port at a time.

## 3. Board: one-time U-Boot environment

Open the console (`sudo minicom -D /dev/ttyUSB0 -b 115200`). The stock U-Boot has a 0-second keyed autoboot, so to get the prompt the first time, hold the stop key **from power-on**: `s` on current SpacemiT builds, Space on older ones (the "Autoboot in 0 seconds, press … to stop" line names it). Power-cycle with `tuya-power` while holding the key until `=>` appears. Then:

```
setenv bootcmd 'echo LAVA'
setenv ethaddr 02:11:22:33:44:01
saveenv
reset
```

After `reset` the board must print the U-Boot banner, then `LAVA`, and sit at `=>` with no key pressed. That is the behaviour LAVA relies on. `ethaddr` gives the board a fixed MAC (U-Boot uses a random one otherwise), so its DHCP lease stays stable; U-Boot also writes it into the DTB it boots, so the kernel gets the same MAC.

If `saveenv` answers "Saving Environment to nowhere… not possible", stop here — the env cannot be persisted on that build and the template needs a different interrupt strategy.

Going back to the USB Ubuntu for hands-on work (the `rv2-install-on-board.sh` flow) is one command at the prompt, since only `bootcmd` changed and the vendor `autoboot` script is still in the env:

```
run autoboot
```

To hand the board back to the old behaviour permanently: `setenv bootcmd 'run autoboot; echo "run autoboot"'` then `saveenv`. Take the device offline in LAVA first (`sudo lava-server manage devices update --health MAINTENANCE orangepi-rv2-1`) so a health check doesn't power-cycle it under you.

## 4. Host: board link, firewall, dispatcher IP, staging directory

The boards hang off the host's second USB NIC (`enx00aa3458890a`), an isolated link with no DHCP server. Give it `192.168.2.2/24` (persistently, via NetworkManager), open ufw on that interface only (it defaults to deny; this covers TFTP now and NFS/rpcbind later without touching the office side), tell LAVA that this is the dispatcher address boards should use, and create the staging directory (owned by your user so the build script can `scp` straight into it).

```
sudo nmcli con add type ethernet ifname enx00aa3458890a con-name lab-boards ipv4.method manual ipv4.addresses 192.168.2.2/24 ipv6.method disabled
sudo nmcli con up lab-boards
sudo ufw allow in on enx00aa3458890a from 192.168.2.0/24
sudo mkdir -p /etc/lava-server/dispatcher.d/worker-2 /srv/lava/rv2
sudo chown ali:ali /srv/lava/rv2
echo 'dispatcher_ip: 192.168.2.2' | sudo tee /etc/lava-server/dispatcher.d/worker-2/dispatcher.yaml
ip -4 -br addr show enx00aa3458890a
grep TFTP_DIRECTORY /etc/default/tftpd-hpa
```

(Without NetworkManager: `/etc/network/interfaces.d/lab-boards` with `auto enx00aa3458890a`, `iface enx00aa3458890a inet static`, `address 192.168.2.2/24`, then `sudo ifup enx00aa3458890a`.) The `ip` line must show `192.168.2.2/24`.

The last line shows the TFTP root (`/srv/tftp` on this host). LAVA reads that same file and creates its per-job directories under it, so it needs no change; step 6 copies files into it.

## 5. Boot inputs

Three files in `/srv/lava/rv2/`:

- `Image` and `k1-orangepi-rv2.dtb` — the same mainline/for-next build `rv2-build-and-send.sh` already produces for the board, shipped to the LAVA host instead. `rv2-build-and-send-lava.sh` (next to it in `linux-kernel-notes/scripts/kernel_build_scripts/opirv2/`) does exactly that: same `.config`, same SpacemiT toolchain, same DTB, `Image dtbs` (no kernel modules — the buildroot ramdisk loads none) plus the kselftest collections in `KSELFTEST_TARGETS` packed as `kselftest.tar.xz`, then `scp` into `/srv/lava/rv2/`. On the workstation, inside the kernel checkout:

  ```
  # first time only: the .config. Your existing rv2-build-and-send.sh .config works as-is;
  # from scratch, defconfig + the K1 EMAC/PCIe symbols (ramdisk boots need only defconfig):
  make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- defconfig
  scripts/config -e SPACEMIT_K1_EMAC -e PCIE_SPACEMIT_K1 -e PHY_SPACEMIT_K1_PCIE -e PHY_SPACEMIT_K1_USB2
  make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- olddefconfig

  # every build:
  ~/.WORKDIR/linux-kernel-notes/scripts/kernel_build_scripts/opirv2/rv2-build-and-send-lava.sh
  ```

  The script refuses to build without the `ARCH_SPACEMIT`/CCU/reset/pinctrl/GPIO symbols and warns if `SPACEMIT_K1_EMAC` is not built in (fine for ramdisk jobs, fatal for NFS-root ones). It leaves `KVER` in `/srv/lava/rv2/` so you can tell which build is staged. `LAVA_HOST`, `LAVA_DIR`, `CROSS` are the config block at the top.

  The DTB name is the upstream one your board already boots with (`fdtfile=spacemit/k1-orangepi-rv2.dtb` in `orangepiEnv.txt` after `rv2-install-on-board.sh`), not the BSP `ky/x1_orangepi-rv2.dtb`; the LAVA template's `booti` line matches the manual `booti` that works on this board.

- `rootfs.cpio.gz`: KernelCI's buildroot-baseline initramfs for riscv (busybox, prompt `~ #`, the same image KernelCI's own baseline jobs use). On the host:

  ```
  sudo wget -O /srv/lava/rv2/rootfs.cpio.gz https://storage.kernelci.org/images/rootfs/buildroot/buildroot-baseline/20260915.0/riscv/rootfs.cpio.gz
  ```

  (`20260915.0` is the version KernelCI pinned in September 2026; if it 404s, browse `https://storage.kernelci.org/images/rootfs/buildroot/buildroot-baseline/` for the newest version and its `riscv/` directory.)

```
ls -l /srv/lava/rv2/
```

## 6. Manual netboot test (before involving LAVA)

This proves the stock U-Boot's Ethernet and TFTP work (the RV2 has Motorcomm PHYs and the vendor U-Boot only carries the Realtek PHY driver, so it runs on the generic PHY driver — verified: link comes up at 100 Mbit full duplex) and that the kernel/DTB pair boots. The board link is isolated with no DHCP server, so the board uses a static address, exactly like the BPI-F3 template. (Diagnosed with `tcpdump` and `ip neigh`: the board's ARP showed up on `enx00aa3458890a`, not on the office-LAN NIC.)

```
sudo cp /srv/lava/rv2/Image /srv/lava/rv2/k1-orangepi-rv2.dtb /srv/lava/rv2/rootfs.cpio.gz /srv/tftp/
```

At `=>` on the console:

```
setenv ipaddr 192.168.2.6
setenv netmask 255.255.255.0
setenv serverip 192.168.2.2
ping 192.168.2.2
setenv fdt_high 0xffffffffffffffff
setenv initrd_high 0xffffffffffffffff
tftpboot 0x11000000 Image
tftpboot 0x31000000 k1-orangepi-rv2.dtb
tftpboot 0x21000000 rootfs.cpio.gz
setenv bootargs 'console=ttyS0,115200n8 earlycon=sbi root=/dev/ram0 rw'
booti 0x11000000 0x21000000:${filesize} 0x31000000
```

Expected: `ping` says `host 192.168.2.2 is alive`, each `tftpboot` prints `Bytes transferred = …`, and the kernel boots to a `~ #` prompt (buildroot starts in `/root`). Afterwards remove the copies from the TFTP root (`sudo rm /srv/tftp/{Image,k1-orangepi-rv2.dtb,rootfs.cpio.gz}`).

If `ping` fails, the board side (cable, switch port — U-Boot only drives the port it calls `ethernet@cac80000`) needs attention before anything else; if `ping` works but `tftpboot` times out, it is the host firewall or the TFTP root.

## 7. LAVA device type and device

Create `/etc/lava-server/dispatcher-config/device-types/orangepi-rv2.jinja2` with the content below. The three `tuya-power … 2` lines carry the outlet number from step 1.

```jinja
{% extends 'base-uboot.jinja2' %}

{% set connection_list = ['uart0'] %}
{% set connection_commands = {'uart0': 'telnet localhost 5002'} %}
{% set connection_tags = {'uart0': ['primary', 'telnet']} %}

{% set power_off_command = "/usr/local/bin/tuya-power off 2" %}
{% set power_on_command = "/usr/local/bin/tuya-power on 2" %}
{% set hard_reset_command = "sh -c '/usr/local/bin/tuya-power off 2; sleep 5; /usr/local/bin/tuya-power on 2'" %}
{% set bootloader_prompt = '=>' %}
{% set bootloader_terminal = 'uart0' %}
{% set extra_boot_methods = '' %}

# Stock SpacemiT U-Boot (2022.10ky) in the on-board NOR; its saved env has
# bootcmd='echo LAVA', so like the BPI-F3 it drops straight to the prompt.
{% set interrupt_prompt = '=>' %}
{% set uboot_needs_interrupt = False %}

# Load addresses (same DDR layout as the BPI-F3). Setting booti_* makes LAVA
# boot the plain Image with booti instead of wrapping it into a uImage.
{% set booti_kernel_addr = '0x11000000' %}
{% set booti_ramdisk_addr = '0x21000000' %}
{% set booti_dtb_addr = '0x31000000' %}
{% set uboot_mkimage_arch = 'riscv' %}

# 64-bit all-Fs, or the FDT/initrd relocation fails on this board.
{% set uboot_fdt_high = '0xffffffffffffffff' %}
{% set uboot_initrd_high = '0xffffffffffffffff' %}

# Isolated board link, no DHCP: static address. serverip comes from
# dispatcher_ip (192.168.2.2) via {SERVER_IP}.
{% set uboot_ipaddr_cmd = 'setenv ipaddr 192.168.2.6; setenv netmask 255.255.255.0' %}
{% set base_ip_args = 'ip=192.168.2.6:192.168.2.2::255.255.255.0::eth0:off' %}

# 'tftp' is ambiguous on this U-Boot (tftpput/tftpsrv are built in): spell out tftpboot.
{% set uboot_tftp_commands = [
    "tftpboot {KERNEL_ADDR} {KERNEL}",
    "tftpboot {RAMDISK_ADDR} {RAMDISK}",
    "setenv initrd_size ${filesize}",
    "tftpboot {DTB_ADDR} {DTB}"
] %}

# net.ifnames=0 keeps the NIC named eth0 under the Debian NFS initrd (see step 10).
{% set base_kernel_args = 'console=ttyS0,115200n8 earlycon=sbi net.ifnames=0' %}
```

Register the device type and the device, give the device a dictionary that just extends the type, and review what LAVA rendered (the render comes from the server via lavacli; there is no `device-dictionary` management command in this LAVA version):

```
sudo lava-server manage device-types add orangepi-rv2
sudo lava-server manage devices add --device-type orangepi-rv2 --worker worker-2 orangepi-rv2-1
echo "{% extends 'orangepi-rv2.jinja2' %}" | sudo tee /etc/lava-server/dispatcher-config/devices/orangepi-rv2-1.jinja2
sudo lava-server manage devices update --health UNKNOWN orangepi-rv2-1
sudo lava-server manage devices details orangepi-rv2-1
sudo lavacli -i IDENTITY devices dict get --render orangepi-rv2-1 | grep -nE 'tftpboot|booti|tuya|telnet|needs_interrupt'
```

`details` must show `device-dict: True`, and the grep must echo the tftp commands, the three `tuya-power` lines, `telnet localhost 5002` and `needs_interrupt: False`.

## 8. Health-check job

Save as `orangepi-rv2-healthcheck.yaml`:

```yaml
device_type: orangepi-rv2
job_name: orangepi-rv2 health check (tftp ramdisk boot)
priority: medium
visibility: public

timeouts:
  job:
    minutes: 15
  action:
    minutes: 5
  connection:
    minutes: 2

actions:
- deploy:
    timeout:
      minutes: 5
    to: tftp
    os: oe
    kernel:
      url: file:///srv/lava/rv2/Image
      type: image
    dtb:
      url: file:///srv/lava/rv2/k1-orangepi-rv2.dtb
    ramdisk:
      url: file:///srv/lava/rv2/rootfs.cpio.gz
      compression: gz

- boot:
    timeout:
      minutes: 5
    method: u-boot
    commands: ramdisk
    prompts:
    - '~ #'

- test:
    timeout:
      minutes: 5
    definitions:
    - repository: https://github.com/Linaro/test-definitions
      from: git
      path: automated/linux/smoke/smoke.yaml
      name: smoke-tests
      params:
        TESTS: "pwd, uname -a, ip a, lscpu, vmstat, lsblk"
```

(`lsb_release` is left out of the smoke list because buildroot has no such command.)

```
lavacli -i local jobs submit --follow ~/orangepi-rv2-healthcheck.yaml
```

It should power-cycle the board, see `=>`, run the U-Boot command list (compare it with the render from step 7), reach `~ #`, and pass the smoke tests — job 299 on 2026-09-20 was the first green run. Then install it as the device type's health check (a file per device type; setting the device's health to Unknown triggers it and the device goes `Good` on pass):

```
sudo cp ~/orangepi-rv2-healthcheck.yaml /etc/lava-server/dispatcher-config/health-checks/orangepi-rv2.yaml
sudo lava-server manage devices update --health UNKNOWN orangepi-rv2-1
```

## 9. If something fails

- Job stuck at "waiting for prompt `=>`": open `telnet localhost 5002` and power-cycle by hand — either ser2net is on the wrong adapter, or the board's `bootcmd` did not persist (step 3).
- `tftpboot` fails inside the job but worked manually: `ip -4 -br addr show enx00aa3458890a` must still show `192.168.2.2/24` (the NetworkManager connection must be up), and the `setenv ipaddr`/`netmask`/`serverip` lines in the template must match what you typed by hand.
- `tftpboot` times out: `sudo ufw status` must show the `Anywhere on enx00aa3458890a` rule from `192.168.2.0/24`; `/etc/default/tftpd-hpa` must point at `/srv/tftp`.
- Kernel boots but no `~ #`: wrong ramdisk (needs a busybox/buildroot initramfs, not an Ubuntu `initrd.img`) or the prompt differs — read the console log in the job and adjust `prompts`.
- Power step fails: run `sudo tuya-power status` — a "subscription expired" or connect error is the Tuya cloud side.

## 10. NFS root and kselftest

The buildroot ramdisk cannot hold the kselftest tree, so kselftest jobs boot a KernelCI Debian rootfs over NFS. Job 299's log settles the two unknowns: the cabled port is `eth0` under mainline and the static `ip=` from `base_ip_args` configures it, so the template's `nfs` method works as rendered. What is needed on top:

Host: the NFS server (LAVA extracts the rootfs per job and exports it itself; the ufw rule from step 4 already admits NFS/rpcbind from the board link), the KernelCI Debian `trixie-kselftest` riscv64 rootfs plus its `initrd.cpio.gz` (KernelCI boots NFS jobs with that small initrd, which mounts the NFS root), and one template line pinning NFSv3 — riscv `defconfig` has `NFS_V2=y`, which makes nfsroot default to v2, and Debian's server has v2 switched off:

```
sudo apt install -y nfs-kernel-server
wget -O /srv/lava/rv2/trixie-kselftest.rootfs.tar.xz https://storage.kernelci.org/images/rootfs/debian/trixie-kselftest/20260915.0/riscv64/full.rootfs.tar.xz
wget -O /srv/lava/rv2/trixie-kselftest.initrd.cpio.gz https://storage.kernelci.org/images/rootfs/debian/trixie-kselftest/20260915.0/riscv64/initrd.cpio.gz
echo "{% set base_nfsroot_args = 'nfsroot={NFS_SERVER_IP}:{NFSROOTFS},vers=3,tcp,hard' %}" | sudo tee -a /etc/lava-server/dispatcher-config/device-types/orangepi-rv2.jinja2
```

(If a `wget` 404s, browse `https://storage.kernelci.org/images/rootfs/debian/trixie-kselftest/` for the current version directory.)

Workstation: `rv2-build-and-send-lava.sh` (step 5) also builds the kselftest collections named in its `KSELFTEST_TARGETS` (default `size dt exec`), packs them under `opt/kselftest/` and ships `kselftest.tar.xz` next to the kernel. The board has no route to the internet, so the tarball rides into the NFS root through the deploy's `modules:` field, which simply extracts a tarball into the rootfs. Set `KSELFTEST_TARGETS=""` for a kernel-only run.

`dt` is worth having first: it reports every DT node whose compatible has no driver bound — the unowned-strip measurement, taken on the board.

Job (`orangepi-rv2-kselftest.yaml`): `deploy: to: tftp` with `os: debian`, kernel, dtb, `ramdisk:` = the initrd, `nfsrootfs:` = the rootfs tarball, `modules:` = the kselftest tarball; `boot: commands: nfs` with `auto_login: {login_prompt: 'login:', username: root}` and prompt `/ # `; test = Linaro `automated/linux/kselftest/kselftest.yaml` with `SKIP_INSTALL: "true"`, `KSELFTEST_PATH: /opt/kselftest`, `TST_CMDFILES: "size dt exec"`, `SKIPFILE: "none"` (space-separated `TST_CMDFILES`: the definition loops `for test in ${TST_CMDFILES}` and greps `^<collection>:` in `kselftest-list.txt`, so a comma-joined list matches nothing; each KTAP result is reported as a LAVA test case).

Three things that this NFS/Debian path needs, all verified against jobs 302–303:

- **Prompt.** The `trixie-kselftest` rootfs auto-logs root into a busybox-style shell whose prompt is `/ # `, not bash's `root@host:~#`. The boot action must list `/ # ` in `prompts:` or `login-action` times out ~9 min after a fully successful boot and NFS mount. (The job keeps `root@(.*):[/~]#` as a second pattern for forward-compat.)
- **`net.ifnames=0`** in `base_kernel_args` (device-type template). Without it the Debian initrd's systemd-udev renames `eth0` -> `end0` mid-boot; the kernel `ip=...:eth0:off` config and the initramfs NFS scripts both key on `eth0`, so the initramfs spends ~3 min in "Waiting up to 180 secs for eth0 to become available / SIOCGIFINDEX: No such device" before falling through. The root still mounts (kernel-level IP-Config brought the link up at 2.7 s), but the stall is pure waste and eats the boot timeout. `net.ifnames=0` keeps the interface named `eth0` end to end. Harmless to the buildroot health check (no systemd, no rename).
- **`SKIPFILE: "none"`.** `kselftest.yaml` always passes `-S "${SKIPFILE}"` and defaults it to `""`; `kselftest.sh`'s `-S` handler tests `[ -z "${OPTARG##*http*}" ]`, which is *true* for an empty string, so it treats `''` as an http URL, runs `wget ''` ("Prepended http:// to ''", "Invalid host name"), and `exit 1`s during option parsing — before it ever looks for `/opt/kselftest`. Any non-empty token that contains neither `http` nor a `.yaml` suffix routes to the plain-skipfile branch instead; the file (`<def-dir>/none`) doesn't exist, so the later `[ -f "${SKIPFILE}" ]` is false and no skips are applied. This is an upstream bug in the Linaro definition (worth a one-line fix: guard the empty case).

Reading the results: a `shardfile-<collection>: fail` test case means that collection has no `^<collection>:` lines in `kselftest-list.txt` — its tests never cross-compiled/installed on the workstation, so it isn't in the tarball at all (`kselftest-install` silently keeps going past per-target build failures). `rv2-build-and-send-lava.sh` warns about this at pack time. The `dt` collection's per-node fails are the *measurement*, not an infra problem: each `fail` is a DT node whose compatible has no driver bound in the running kernel — the RV2's mainline driver-gap list, and the overall `dt_test_unprobed_devices_sh` case stays `fail` until every node probes (or a board skipfile encodes the known gaps so only regressions stand out).

```
lavacli -i local jobs submit --follow ~/orangepi-rv2-kselftest.yaml
```

- **KernelCI**: add a platform `spacemit-k1-orangepi-rv2` (arch `riscv`, `boot_method: u-boot`, `mach: spacemit`, `dtb: dtbs/spacemit/k1-orangepi-rv2.dtb`, compatible `xunlong,orangepi-rv2` / `spacemit,k1`) next to the existing `spacemit-k1-bananapi-f3` entry, and a scheduler entry for this lab's runtime.
