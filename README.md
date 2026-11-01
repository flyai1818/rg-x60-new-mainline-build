# RG-X60 New Mainline Build

GitHub Actions build for the Ruijie RG-X60 New (MT7986A, 512 MiB RAM, 128 MiB SPI-NAND, EN8811H 2.5G WAN PHY), built on the official `immortalwrt/immortalwrt` `master` branch.

## Scope

- Builds against the current official `immortalwrt/immortalwrt` `master` branch (Linux 6.18).
- Adds only the RG-X60 New board-support layer needed for the fixed MTK U-Boot layout.
- Uses the fixed UBI partition: offset `0x680000`, size `0x6b00000` (107 MiB).
- Does **not** build, download, or modify BL2 or FIP.
- Keeps native nftables/firewall4, MediaTek HNAT, EN8811H, PPPoE, Wi-Fi, procd-ujail and debugfs.
- Enables eBPF kernel integration (cgroup-BPF, kprobes, BTF, BPF stream parser, XDP sockets, Netkit), TPROXY, TUN and veth, with BBR and FQ as defaults.
- Removes the optional LuCI **Plugins** and **Network Diagnostics** pages (`scripts/remove-unused-luci-pages.sh`); their core LuCI administration modules remain intact.
- On a fresh configuration, radios without an existing country setting receive `AU` (`files/etc/uci-defaults/99-rg-x60-wireless-country`). A retained or manually set country is never overwritten.
- Does not embed DAE, Honk or sing-box cores or LuCI applications.

## Single firmware profile

There is no flavor selection. Every build uses the same profile, defined in `defconfig/mainline-rg-x60-new-107m.config`.

Legacy iptables is excluded; firewall4 uses native nftables only.

## TCP tuning

The build writes `files/etc/sysctl.d/90-tcp-tuning.conf` into the rootfs:

- BBR congestion control and FQ queueing by default.
- 32 MiB maximum socket buffers (`rmem_max`, `wmem_max`, `tcp_rmem`, `tcp_wmem`).
- `netdev_max_backlog = 10000`.

The "Verify final kernel options" step checks that these lines are present in the rootfs, so a build fails if the file is missing.

## Build

Open **Actions** → **Build ImmortalWrt Mainline RG-X60 New (107MiB)** → **Run workflow**. Inputs:

- `clean_build` — ignore caches and rebuild from scratch.
- `ssh_debug` — open an SSH session on failure.

The artifact `RG-X60-New-mainline-107M-<commit>` contains:

- `RG-X60-New-mainline-107M-sysupgrade.bin` — the normal upgrade image, for LuCI or `sysupgrade`.
- `mainline-rg-x60-new-107m.config.full` — the resolved configuration, for reproducing the build.
- `BUILD-INFO`, `manifest` and `kernel.config` — build metadata and the final kernel configuration.

## Caching

- Source downloads (`dl/`) are cached per commit.
- The host tools and toolchain (`staging_dir/host`, `hostpkg`, `toolchain-*`) are cached per commit and defconfig. On a cache hit, these steps are skipped.
- ccache is capped at 4 GB. The repository cache limit is 10 GB, and the caches compete for it.
- Cache save steps are non-fatal (`continue-on-error`), so a cache problem cannot prevent the firmware from being uploaded.

## Known issues

- **EEE:** the `eth1` EEE-off hooks (`etc/rc.local` and `etc/hotplug.d/net/60-eth1-eee-off`) are ineffective. The EN8811H path reports `Cannot get EEE settings: Not supported`, so `ethtool --set-eee` has no effect. They are harmless and are kept for now.
- **Single-flow proxy throughput:** iperf3 over the WAN path reaches about 400 Mbps after the TCP buffer change. Proxy throughput must be verified on the router with the same node and client before it is considered resolved.

## Status

Before compiling, the workflow resolves the configuration with `make defconfig` and checks the selected kernel and package options. After compiling, it verifies the kernel options, the rootfs sysctl and boot hooks, and requires both flashable images before uploading.

Hardware installation and runtime testing are still required before treating an artifact as production-ready.
