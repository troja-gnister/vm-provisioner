# vmp: Python Rewrite Design

**Date:** 2026-10-04
**Status:** Approved in brainstorming; pending written-spec review
**Replaces:** the Rust `vm-provisioner` (last Rust commit to be tagged `rust-final`)

## 1. Summary

Rewrite vm-provisioner in Python as **vmp**, a Qubes/Whonix-style compartmentalization tool for a single-user Fedora Atomic (Silverblue) + GNOME desktop. Each app runs in its own KVM VM, and its windows appear on the host desktop with a trust-color border. VMs boot from shared, verified templates. Network access is routed through gateway VMs (Mullvad over WireGuard, or Whonix-Gateway for Tor) and enforced outside the guest. Microphone access is impossible by design. Clipboard and file movement between domains is explicit and approved on the host. There is a CLI (`vmp`) and a GTK4/libadwaita manager ("VM Manager"), both thin front ends over one core library.

This is not a 1:1 port. Features are redesigned for security and usability, and several are dropped (§15).

## 2. Goals and non-goals

### Goals
- **G1 Host isolation:** a compromised guest cannot reach the host except through the hypervisor and the xpra client (accepted residual risk, §4).
- **G2 Inter-VM isolation:** VMs cannot reach each other over the network, and they share data only through host-mediated, user-approved actions.
- **G3 No microphone:** no VM can capture host audio input. Speaker output works.
- **G4 Enforced network routing:** a VM's network path (none / direct / VPN / Tor) is enforced by the host and gateway VMs, not by the guest. If the tunnel fails, traffic stops; it never falls back to a direct connection.
- **G5 No clipboard leakage:** host clipboard contents never reach a VM without an explicit user action.
- **G6 Mullvad-proof display:** windows and audio never break because of host VPN state.
- **G7 Usability:** a new app VM takes seconds, disposables take one click, apps appear in the GNOME app grid, and `vmp doctor` explains every setup problem.
- **G8 Minimal privilege:** one bounded root step at setup. Day-to-day operation needs only `libvirt` group membership. No passwords exist anywhere.

### Non-goals
- Protecting against a compromised host, hypervisor/QEMU 0-days, or physical attacks.
- Anonymity against browser fingerprinting for Fedora VMs behind Tor (Tor Browser is recommended in the VM; the manager shows this note).
- A custom remote-display protocol (xpra stays, behind an interface).
- Non-Fedora app templates, GPU/PCI passthrough, CPU pinning, shared folders, web streaming (§15).
- A background daemon (architecture A; the core is designed so a daemon could be added later).

## 3. Decisions log

| Topic | Decision |
|---|---|
| Language | Python 3.14 (host system Python); no pip-installed runtime deps |
| Threat model | Compartmentalization and network privacy, weighted equally |
| Architecture | Shared `core` library + CLI + GTK4 manager; no daemon; libvirt is the source of truth |
| libvirt access | `python3-libvirt` bindings, `qemu:///system`, via `libvirt` group membership |
| VM model | Qubes-style: template root reset every boot + persistent private disk; disposables; gateways |
| Template source | Fedora Cloud Base qcow2, GPG-verified CHECKSUM, provisioned once via a cloud-init seed |
| In-guest admin | `user` has no password and no sudo; system changes are host-driven through qemu-guest-agent |
| Per-VM identity | JSON over libvirt `fw_cfg`, applied by `vmp-init` every boot |
| Display | xpra over **native vsock**: no SSH, no TCP; security flags enforced on the host client |
| Audio | xpra speaker forwarding; client `--microphone=off`; no emulated sound card |
| Clipboard | xpra clipboard **off**; explicit push (host→VM) and pull (VM→host, approved) via shortcuts |
| Files | Host-mediated copy via qemu-guest-agent into fixed `Incoming` dirs; approval for VM-sourced files |
| VPN | Mullvad over plain WireGuard inside `sys-vpn` gateway VM(s), with an nftables kill switch |
| Tor | Official Whonix-Gateway KVM image as `sys-tor` |
| USB | Host-owned controllers; one VM per device at a time; audio-interface and boot-HID devices refused by default |
| Interface | CLI + libadwaita manager, built together phase by phase |
| Names | CLI `vmp`, app "VM Manager", same repository |

## 4. Threat model

**Trusted:** host OS, host user session, libvirt/QEMU (as shipped by Fedora), the vmp code, Fedora and Whonix signing keys.

**Untrusted:** everything inside every VM, including templates after first boot, since packages installed into a template could be malicious. This covers all data a guest emits: window contents, titles, `.desktop` files, icons, clipboard data, files, qemu-guest-agent responses, and network traffic.

**Attack surfaces kept deliberately small:**
| Surface | Mitigation |
|---|---|
| xpra client parsing guest data | Accepted residual risk. Host client runs with every optional feature disabled (§9.1). Revisit if xpra has CVEs; the display backend is behind `DisplayBridge`. |
| qemu-guest-agent replies | Parsed by libvirt (JSON). vmp enforces size limits and timeouts on every call. |
| Guest-provided metadata (`.desktop`, icons) | Strict parser, length limits, icons re-encoded with Pillow; never placed in host command lines. |
| Network to host | `vmp-uplink` reaches only host DNS (firewalld zone `vmp`); downstream networks have no host address at all. |
| Spoofing between VMs | Isolated bridge ports + libvirt `clean-traffic` nwfilter + static addressing. |
| UI spoofing | Host-enforced `[vm]` title prefix and label-color border; guest notifications/tray disabled; approval dialogs only in host GTK windows. |

## 5. Concepts

| Type | Description | What persists |
|---|---|---|
| **Template** (`fedora-44`) | OS, system packages, system Flatpaks. The only place system software is installed. | Whole disk; changed only by `template update/install/shell`. |
| **App VM** (`work`) | Boots the template root as a transient overlay, plus its own private disk at `/rw`. | `/rw` only (`/home` is bind-mounted from `/rw/home`). |
| **Disposable** (`disp-<4 digits>`) | Transient libvirt domain from a template (optionally on top of a stopped base app VM's private disk). Runs one app, powers off when it exits. | Nothing. |
| **Gateway** (`sys-vpn`, `sys-tor`) | Service VM providing network to other VMs. VPN gateways are app VMs built from the Fedora template with `role=gateway`. `sys-tor` is a Whonix-Gateway import. | VPN: `/rw/config`. Whonix: its whole disk (standalone). |

**Labels:** `red`, `orange`, `yellow`, `green`, `blue`, `purple`, `gray`, `black`. Each maps to a fixed RGB value. A label drives the xpra border color, the launcher icon tint, the manager color dot, and the `[name]` window-title prefix.

**Names:** `^[a-z][a-z0-9-]{0,30}$`, checked in `core/names.py` before a name reaches any path, XML, or command. `disp-` and `sys-` prefixes are reserved for their types.

**Per-VM settings** (stored in libvirt metadata, §12.2):
| Key | Type / values | Default |
|---|---|---|
| `label` | label enum | `blue` (app), `red` (disposable), `black` (gateway) |
| `template` | template name | global `default_template` |
| `network` | `none` / `direct` / `<gateway name>` | global `default_network` (default `none`; the user changes it, e.g. to `sys-vpn`, once a gateway exists) |
| `memory_mb` | int ≥ 512 | 2048 |
| `vcpus` | int ≥ 1 | 2 |
| `private_size_gb` | int, grow-only | 10 |
| `apps` | list of desktop IDs exposed to the host | `[]` |
| `usb_allow` | list of `{vendor, product, serial?, allow_mic?, force_hid?}` | `[]` |

## 6. Storage and templates

### 6.1 Layout
libvirt storage pool `vmp` at `/var/lib/libvirt/images/vmp/` (SELinux `virt_image_t` context, created once by `vmp setup`; volumes are managed through the libvirt API with no sudo):
```
vmp/templates/<template>/root.qcow2    # 20G sparse
vmp/vms/<vm>/private.qcow2             # private_size_gb sparse
vmp/whonix/sys-tor/root.qcow2          # imported Whonix-Gateway disk
```

### 6.2 Disks per VM type
| Type | Root | Private |
|---|---|---|
| Template | `root.qcow2` read-write | none |
| App VM / Gateway (VPN) | template `root.qcow2` with `<transient shareBacking='yes'/>` | `private.qcow2`, serial `vmp-private` |
| Disposable | template root, transient | blank transient volume, or base VM `private.qcow2` as transient (base must be stopped) |

The template must be stopped while any dependent VM runs, and the reverse. QEMU image locking enforces this; `core/lifecycle.py` checks first and raises `PolicyDenied("stop work, banking first")`. Fallback if transient `shareBacking` fails (spike S3): vmp creates per-boot overlay volumes with a libvirt `backingStore`, deletes them after shutdown, and removes stale overlays before the next start.

### 6.3 Guest units baked into the template
- **`vmp-init.service`** (Python, early boot): reads `/sys/firmware/qemu_fw_cfg/by_name/opt/vmp/config/raw`, validates it against the schema (§6.5), then applies hostname, static network config, and role-specific setup.
- **`vmp-private.service`**: if `/dev/disk/by-id/virtio-vmp-private` has no filesystem, runs `mkfs.ext4 -L vmp-private`. Mounts it at `/rw`, bind-mounts `/rw/home` to `/home`, and seeds `/rw/home/user` from `/etc/skel` on first use. Skipped for templates.
- **`vmp-xpra.service`** (user service, lingering enabled for `user`): `xpra start :100 --bind-vsock=auto:14500 --mdns=no --daemon=no`, with no TCP or Unix-socket binds beyond xpra's local control socket, and clipboard, file transfer, printing, webcam, and notifications disabled on the server side too (defense in depth; the host client flags are what count). The unit reads `/run/vmp/xpra.env`, which root-owned `vmp-init` renders from fw_cfg (unprivileged `user` cannot read fw_cfg directly).
- **Disposable role:** `vmp-init` adds `--start-child=<app> --exit-with-children` to `xpra.env`, and the unit's `ExecStopPost=` powers the VM off. Because the domain is transient, libvirt removes it and its transient disks.
- **Gateway role (`gateway-vpn`):** `vmp-gateway.target` pulls in the nftables kill-switch ruleset (§7.3), `wg-quick@wg0` (config at `/rw/config/wg0.conf`), and `dnsmasq` for downstream DNS. The target is reached only if all three start; the host treats `systemctl is-active vmp-gateway.target` == `active` (via qemu-guest-agent) as the gateway's ready signal.

### 6.4 Template build (`vmp template create fedora-44`)
1. Download `Fedora-Cloud-Base-Generic-44-*.x86_64.qcow2` and the matching `CHECKSUM` into `~/.cache/vmp/images/`.
2. `gpgv --keyring <tmp keyring from /etc/pki/rpm-gpg/RPM-GPG-KEY-fedora-44-primary> CHECKSUM`; then check the image's SHA-256 against the signed file. On any failure: `VerificationFailed`, delete the download, stop.
3. Create the volume in the pool, `virStorageVolUpload` the image, resize to 20G.
4. Build a NoCloud seed ISO with `xorrisofs -V cidata` (in `mkdtemp` under `$XDG_RUNTIME_DIR`) containing `user-data` that runs the provisioning script:
   - install `xpra`, `qemu-guest-agent`, `flatpak` (+ Flathub remote), `pipewire` + `pipewire-pulseaudio` + `wireplumber`, `xclip`, `nftables`, `wireguard-tools`, `dnsmasq`
   - install the vmp guest units (§6.3) and nftables/dnsmasq templates for the gateway role
   - harden: SELinux enforcing; root locked; create `user` with no password, no sudo, no wheel; remove `openssh-server`; disable `sshd`, `cups`, `bluetooth`; enable `qemu-guest-agent`, `vmp-init`, `vmp-private`
   - `dnf remove -y cloud-init` then `poweroff`
5. Boot once with a temporary NIC on `vmp-uplink` and the seed attached. The seed's `network-config` (v2) gives the NIC a static address, because `vmp-uplink` has no DHCP. Wait for power-off (timeout 30 min), then detach the seed and remove the NIC from the template definition.

### 6.5 fw_cfg config schema (version 1)
```json
{
  "version": 1,
  "name": "work",
  "role": "template | app | disposable | gateway-vpn",
  "label": "blue",
  "network": {"mode": "none | static", "address": "10.137.0.12/24", "gateway": "10.137.0.1", "dns": ["10.137.0.1"]},
  "xpra": {"vsock_port": 14500},
  "disposable": {"app": "org.mozilla.firefox.desktop", "open_file": "/home/user/Incoming/host/x.pdf"},
  "gateway": {"downstream_address": "10.138.0.1/24"}
}
```
`vmp-init` rejects unknown keys, unknown versions, and invalid values, then boots in `network.mode=none`.

### 6.6 Template administration (host-driven, root via qemu-guest-agent)
- `vmp template update <t>`: checks no dependents are running, attaches the template to the global `update_network` (default: the first VPN gateway, else `direct`), runs `dnf upgrade -y && flatpak update -y` while streaming output, then powers off and removes the NIC.
- `vmp template install <t> <pkg|flatpak-id>...`: same flow, with `dnf install -y` / `flatpak install -y flathub`. Flatpak IDs are recognised by the reverse-DNS pattern.
- `vmp template shell <t>`: starts the template without network and opens a root terminal over xpra.

## 7. Networking

### 7.1 Model
Every VM, gateways included, has `network ∈ {none, direct, <gateway>}`. Chains are allowed (e.g. `sys-tor → sys-vpn → direct`); `core/network.py` rejects loops and self-references. Starting a VM starts its gateway chain first, upstream first.

### 7.2 libvirt networks and addressing
| Network | Kind | Subnet / host IP | Members |
|---|---|---|---|
| `vmp-uplink` | NAT, bridge `vmpbr0`, firewalld zone `vmp` | `10.137.0.0/24`, host `.1` (DNS only; DHCP off) | `direct` VMs, upstream NIC of first-hop VPN gateways, templates during build/update |
| `vmp-net-<vpn-gw>` | L2 only (no `<ip>`, no `<forward>`) | `10.138.0.0/24`, gateway `.1`; same plan reused per gateway (separate L2 segments) | VMs whose `network` is that gateway |
| `vmp-whonix-ext` | NAT, firewalld zone `vmp` | `10.0.2.0/24`, host `.2`, sys-tor `.15` (Whonix KVM defaults; confirmed in S4) | sys-tor upstream when `sys-tor.network=direct` |
| `vmp-net-sys-tor` | L2 only | `10.152.152.0/18`, sys-tor `.10`, clients from `.11` | VMs whose `network` is `sys-tor` |

- Addresses are static, allocated by vmp, recorded in metadata, and delivered by fw_cfg. MACs are random `52:54:00:…`, stored in the domain XML.
- Every VM interface uses `<port isolated='yes'/>` and `<filterref filter='clean-traffic'>` with its `IP`/`MAC`. Gateway downstream interfaces are not isolated, so clients can reach their gateway and nothing else.
- IPv6 is disabled on all downstream networks and inside clients.
- Firewalld zone `vmp`: target `DROP`; allows only `dns` (53/udp, 53/tcp) and ICMP echo to the host. This replaces Fedora's `libvirt` zone, which also allows SSH and TFTP to the host.

### 7.3 VPN gateway (`sys-vpn`, Mullvad)
- `vmp gateway create sys-vpn --kind vpn [--network direct]` creates an app VM with `role=gateway-vpn`, two NICs (upstream + downstream network), and no exposed apps.
- `vmp gateway vpn-import sys-vpn <file.conf>`: the host parses the WireGuard INI. It allows only `[Interface]` keys `PrivateKey, Address, DNS, MTU` and `[Peer]` keys `PublicKey, PresharedKey, AllowedIPs, Endpoint, PersistentKeepalive`. Any `PreUp`, `PostUp`, `PreDown`, `PostDown`, `Table`, `SaveConfig`, or unknown key is rejected with `PolicyDenied`. The checked config is written to `/rw/config/wg0.conf` (mode 0600, root) through qemu-guest-agent, then the gateway restarts. The CLI and manager suggest deleting the source file.
- Inside the gateway (nftables, loaded before any interface comes up; if loading fails, the gateway keeps forwarding disabled and reports `failed` to the host):
  - `forward`: policy drop; allow downstream → `wg0` and established/related return traffic only.
  - `output` on upstream: allow UDP to the configured Endpoint only, plus DNS to the upstream resolver only when the Endpoint is a hostname.
  - `nat`: masquerade on `wg0`; DNAT downstream port 53 (udp/tcp) to the local `dnsmasq`, which forwards only to the tunnel's DNS (default `10.64.0.1`).
- Several VPN gateways are allowed (`sys-vpn-se`, `sys-vpn-us`), each with its own downstream L2 network.

### 7.4 Tor gateway (`sys-tor`, Whonix-Gateway)
- `vmp gateway create sys-tor --kind whonix` downloads the Whonix KVM release, verifies its OpenPGP signature against the Whonix signing key fingerprint pinned in `core/images.py`, extracts **only** the Gateway qcow2, and defines a vmp-generated domain (upstream NIC on `vmp-whonix-ext`, downstream NIC on `vmp-net-sys-tor`, no sound, no USB, no shared folders).
- Clients behind `sys-tor` get `10.152.152.11+`, gateway and DNS `10.152.152.10`.
- Updates run inside Whonix itself (`vmp gateway console sys-tor` opens its console); the manager shows a reminder when the image is older than 30 days.
- **Chaining `sys-tor` behind a VPN gateway** ships only if spike S4 shows Whonix-Gateway's external interface can be reconfigured at import time. Otherwise `vmp` rejects `network=<vpn-gw>` for whonix gateways with a clear error.
- Assigning a VM to `sys-tor` shows the fingerprinting note (§2) once per VM.

### 7.5 Host Mullvad
Display and audio use vsock and are unaffected. VM network traffic leaves through the host's NAT and therefore through, or blocked by, host Mullvad. Spike S2 determines the behavior. The fallback is picked from S2's results and recorded in the findings doc before Phase 3:
- (a) traffic works inside the host tunnel (double hop): no action, documented;
- (b) blocked: add a documented nftables exception for `vmpbr0`, or have `vmp doctor` warn that `sys-vpn` needs host Mullvad off.

## 8. Lifecycle

- `vmp create <name> [--template] [--label] [--network] [--memory] [--vcpus] [--private-size]`: defines the domain, creates `private.qcow2`, allocates addresses. Takes seconds.
- `vmp start|stop|kill|remove <vm>`; `remove` deletes the private disk after a confirmation (`--yes` to skip).
- `vmp set <vm> <key>=<value>` / `vmp show <vm>` / `vmp list`.
- `start` order: resolve the gateway chain, start upstream gateways (waiting for each one's ready signal, §6.3, timeout 120 s), build the fw_cfg JSON, start the domain, wait until qemu-guest-agent answers.
- Domain XML (from `core/domain_xml.py`): q35, UEFI, `host-passthrough` CPU, virtio disks/NICs, `<vsock model='virtio'><cid auto='yes'/></vsock>`, qemu-guest-agent channel, a serial console (`vmp console <vm>`), memballoon on. **No** `<sound>` element, no `<graphics>` (SPICE/VNC), no USB redirection channels. Exception: `sys-tor` may get a graphics console if spike S4 shows Whonix-Gateway has no usable serial console.
- Disposables are created with `createXML` (transient domains), so a power-off removes them completely.

## 9. Display, audio, clipboard, files

### 9.1 Display (xpra over vsock)
- Host client per running VM: `xpra attach vsock://<CID>:14500/` started by `core/display.py`, tracked by a pidfile in `$XDG_RUNTIME_DIR/vmp/<vm>.xpra.pid`. The CID is read from the live domain XML after start.
- Host client flags (enforced; the guest cannot change them): `--microphone=off --webcam=no --speaker=on --clipboard=no --file-transfer=no --open-files=no --open-url=no --printing=no --notifications=no --tray=no --system-tray=no --border=<label-rgb>,3 --title="[<vm>] @title@"`.
- Readiness: poll a vsock connect to `<CID>:14500` (100 ms interval, 60 s timeout → `GuestTimeout`).
- `DisplayBridge` interface: `attach(vm)`, `detach(vm)`, `launch(vm, desktop_id | argv)`, `is_attached(vm)`. xpra is the only implementation.
- Fallback if native xpra vsock fails (spike S1): SSH over vsock (`ProxyCommand` to the vsock port of an in-guest sshd bound only to vsock), using a per-VM key generated by vmp and a host key pinned at first boot via the serial console. Only used if S1 fails.

### 9.2 Apps and launchers
- `vmp apps sync <vm>`: reads `/usr/share/applications/*.desktop`, `/var/lib/flatpak/exports/share/applications/*.desktop`, and `~/.local/share/flatpak/exports/...` through qemu-guest-agent (each file ≤ 64 KiB, ≤ 500 files).
- The parser keeps only `Name` (≤ 64 chars; letters, digits, space, `-_.()+`), the desktop ID (`^[A-Za-z0-9._-]{1,128}\.desktop$`), `NoDisplay`, and the `Icon` name. PNG icons ≤ 1 MiB are fetched, decoded and re-encoded to 64×64 PNG with Pillow, then tinted with the label color. Any other format, or a decode failure, gets the label-colored generic icon.
- `vmp apps expose <vm> <desktop-id>...` writes `~/.local/share/applications/vmp-<vm>-<id>.desktop` with `Name=[<vm>] <Name>` and `Exec=vmp run <vm> --app <desktop-id>`. **Nothing from the guest goes into `Exec`.**
- `vmp run <vm> --app <desktop-id>` starts the VM if needed, attaches the display, then launches through xpra control. `vmp run <vm> -- <argv>` runs a host-typed command.

### 9.3 Audio
No emulated sound card. Guest PipeWire output → xpra speaker forwarding → host client playback. The host client always runs with `--microphone=off`, so no host input is ever captured. The host PulseAudio socket is never shared.

### 9.4 Clipboard
- xpra clipboard is disabled. vmp moves clipboard contents itself, through qemu-guest-agent and `xclip` in the guest on display `:100`.
- `vmp clipboard push [<vm>|--focused]` (Super+Shift+V): host clipboard (UTF-8 text, ≤ 1 MiB) → guest clipboard; shows a notification "Clipboard sent to [vm]". No approval, because the data flows host → VM.
- `vmp clipboard pull [<vm>|--focused]` (Super+Shift+C): guest clipboard → validated (UTF-8, ≤ 1 MiB, NUL removed) → approval dialog (source VM, label color, size, first 500 chars) → host clipboard.
- `--focused`: XWayland `_NET_ACTIVE_WINDOW` → `_NET_WM_PID` → match against xpra client pidfiles → VM. If that fails (spike S5), a GTK chooser of running VMs opens instead.
- Host clipboard read/write happens in a short-lived GTK helper process (the approval dialog for pull, an invisible helper window for push), because on Wayland only a focused client may access the clipboard. `wl-clipboard` is the fallback (S5).
- The shortcuts are installed as GNOME custom keybindings by `vmp setup --shortcuts` (and from the manager's preferences).

### 9.5 File transfer
- `vmp copy <src> <dst>` where each side is a host path or `<vm>:<path>`. VM→VM, VM→host, and host→VM are supported; directories are copied recursively (≤ 10 000 entries).
- Data moves in 1 MiB chunks via qemu-guest-agent `guest-file-*`. The total limit is `transfer_max_bytes` (default 1 GiB).
- Destinations are fixed: in VMs `~/Incoming/<source>/` (`<source>` = VM name or `host`), on the host `~/Downloads/vmp-incoming/<vm>/`. Existing names get a ` (n)` suffix; nothing is overwritten.
- File names are reduced to their base name, NFC-normalised, with `/`, NUL, control characters, leading `.` and `..` removed, and limited to 255 bytes. Symlinks are not followed in the source; special files are skipped.
- Every transfer whose source is a VM requires approval (source, destination, file count, total size).
- `vmp open --disposable <host file>` (and the manager action): create a disposable, copy the file into `~/Incoming/host/`, launch `xdg-open` on it as the disposable app; the disposable is destroyed when the app exits.

### 9.6 Approval dialogs
`python3 -m vmp.gui.approve <kind> <json>`: a single-purpose libadwaita dialog. It exits 0 on approve and 1 on deny or close, and the default button is Deny. It is used by the CLI, the keyboard shortcuts, and the manager. Approval never happens in a terminal.

## 10. USB

- `vmp usb list`: enumerates `/sys/bus/usb/devices/*` (bus-port path, vendor, product, serial, interface classes) and shows which VM holds each device (from live domain XMLs).
- `vmp usb attach <vm> <bus-port|vendor:product>` / `vmp usb detach <device>`: live hostdev attach/detach through `attachDeviceFlags(VIR_DOMAIN_AFFECT_LIVE)`. A device attached to another VM is refused (`PolicyDenied`). Devices return to the host when the VM stops.
- If the device is not in the VM's `usb_allow`, an approval dialog asks first, with an "always allow for this VM" option that adds `{vendor, product, serial}`.
- **Refused by default:**
  - any device with an interface of class `0x01` (Audio), including composite webcams. Needs `allow_mic = true` on that allowlist entry.
  - any interface with class `0x03`, subclass `0x01`, protocol `1` or `2` (boot keyboard/mouse). Needs `--force` (or `force_hid = true` in the allowlist).
- usbguard (active on the dev host): behavior defined by spike S6. Planned approach: if usbguard blocks the device, `vmp usb attach` requests a temporary allow through usbguard's D-Bus API (polkit prompt). If that isn't workable, the `doctor` hint tells the user to allow the device in usbguard first.

## 11. Host setup and doctor

### 11.1 Layered packages
`rpm-ostree install libvirt-daemon-kvm qemu-kvm qemu-img edk2-ovmf xpra python3-libvirt xorriso` (+ reboot). `virt-install`, `socat`, and `sshpass` are no longer needed. Already on the host: `gpgv`, `pkexec`, `wl-clipboard`, PyGObject/GTK4/libadwaita, Pillow.

### 11.2 `vmp setup` (idempotent)
1. Check packages; if any are missing, print the exact `rpm-ostree` command and stop.
2. Run `pkexec /usr/bin/python3 -I <venv>/…/vmp_setup_helper.py` once (isolated mode: no environment variables, no user site-packages). The host user session is trusted (§4), so a helper living in the user's venv is acceptable; it is kept under ~150 lines so it is easy to audit. The helper accepts exactly these operations and nothing else:
   - copy the `libvirt` line from `/usr/lib/group` to `/etc/group` if missing; add the invoking user (from `PKEXEC_UID`) to `libvirt`
   - `systemctl enable --now virtqemud.socket virtnetworkd.socket virtstoraged.socket virtnwfilterd.socket`
   - write `/etc/modules-load.d/vmp.conf` (`vhost_vsock`) and `modprobe vhost_vsock`
   - install the firewalld zone `/etc/firewalld/zones/vmp.xml` and reload firewalld
3. As the user: define and start the `vmp` pool, `vmp-uplink`, and `vmp-whonix-ext` (on demand); define the nwfilter bindings; write `~/.config/vmp/config.toml` defaults.
4. `--shortcuts`: install the GNOME clipboard keybindings.

### 11.3 `vmp doctor`
Checks, each with a specific fix hint: `/dev/kvm`, CPU virtualization flags, packages, `libvirt` group membership in the current session, libvirt sockets, `vhost_vsock`, pool, networks, firewalld zone, xpra version ≥ the minimum from S1, the xpra vsock capability, templates present and verified, gateways healthy (kill switch loaded, tunnel up), usbguard state. The manager shows the same checks on first run, with Fix buttons where a fix can be automated.

## 12. Code structure

### 12.1 Layout
```
pyproject.toml
install.sh
src/vmp/
  core/
    names.py  model.py  store.py  conn.py  domain_xml.py  storage.py
    images.py  templates.py  network.py  gateways/{vpn,whonix}.py
    guest.py  fwcfg.py  display.py  apps.py  clipboard.py  transfer.py
    usb.py  lifecycle.py  doctor.py  config.py  errors.py
  cli/        __main__.py + one module per command group
  gui/        app.py (manager), approve.py, chooser.py, widgets/
  guest/      vmp-init (Python), systemd units, nftables + dnsmasq templates, provision.sh
  setup_helper/ vmp_setup_helper.py
tests/        unit/  libvirt/  e2e/
```
Modules depend inward: `cli` and `gui` → `core`; `core` never imports `cli` or `gui`. Each `core` module exposes a small typed API and can be tested on its own.

### 12.2 State
- Per-VM settings: libvirt domain `<metadata><vmp:vm xmlns:vmp="urn:vmp:1">…</vmp:vm></metadata>`, read and written only by `core/store.py`.
- Global config: `~/.config/vmp/config.toml`. Keys: `default_template`, `default_network`, `update_network`, `disposable_base`, `transfer_max_bytes`, `clipboard_max_bytes`, `shortcuts.push`, `shortcuts.pull`. Read with `tomllib`; written by a small writer limited to flat tables of scalars and lists (`core/config.py`).
- Runtime: `$XDG_RUNTIME_DIR/vmp/` (pidfiles, temp dirs, mode 0700). Cache: `~/.cache/vmp/images/`. Logs: `~/.local/state/vmp/vmp.log`.
- No secrets on the host. WireGuard keys exist only on the gateway's private disk after import.

### 12.3 Coding rules
- `subprocess.run([...])` with argument lists only, never `shell=True`; tools are resolved to absolute paths at startup (`shutil.which`) and reported by `doctor`.
- XML is built only with `xml.etree.ElementTree`. Guest-provided data is never parsed as XML.
- Temp files only via `tempfile.mkdtemp(dir=$XDG_RUNTIME_DIR/vmp)`.
- Fail closed: verification, kill-switch, policy, and transport failures raise; there is no fallback to a weaker mode at runtime.
- Clipboard and file contents are never logged.
- Full type hints; `mypy --strict` on `core`; `ruff` for lint and format.

### 12.4 Errors
`VmpError(message, hint)` base class with the subclasses `PrerequisiteMissing`, `NotFound`, `InvalidName`, `InvalidConfig`, `PolicyDenied`, `VerificationFailed`, `GuestTimeout`, `LibvirtFailure` (wraps `libvirt.libvirtError`). CLI exit codes: 0 ok, 1 generic, 2 usage, 3 prerequisite, 4 policy denied / approval denied, 5 verification failed, 6 timeout. The CLI prints `error: …` / `hint: …`; the GUI shows an `AdwAlertDialog` for errors from user actions and an `AdwToast` for background ones.

### 12.5 GTK manager
- `AdwApplication` (`org.vmp.Manager`), `AdwNavigationSplitView`.
- Sidebar: groups Templates / App VMs / Gateways / Disposables; each row shows the label dot, state, and network badge.
- VM page: start/stop/kill; app launch buttons; settings (label, network, memory, vCPUs, private size); USB devices; Send file; Clipboard push/pull.
- Template page: Update, Install, Shell, Dependents.
- New VM dialog: name, label, template, network.
- First run: doctor checklist.
- Events: `libvirt.virEventRegisterDefaultImpl()` with a dedicated thread running `virEventRunDefaultImpl()`; domain lifecycle callbacks are passed to the GTK main loop with `GLib.idle_add`.
- Long operations run in worker threads with progress callbacks, so the GTK main loop is never blocked.

### 12.6 Dependencies and install
- Runtime: stdlib, `python3-libvirt` (layered), PyGObject + GTK4 + libadwaita, Pillow (system).
- Dev: `pytest`, `hypothesis`, `ruff`, `mypy` in `.venv` created with `--system-site-packages`.
- `install.sh`: creates `~/.local/share/vmp/venv` (`--system-site-packages`), installs the package, links `~/.local/bin/vmp`, and installs the manager `.desktop` file. No files are installed outside `$HOME` except by `vmp setup` (§11.2); pkexec uses its default admin-authentication policy, so no custom polkit policy is needed.

## 13. Testing

1. **Unit** (`tests/unit`, no libvirt, every change): names, model validation, XML builders (golden files, then parsed back and checked), WireGuard validator, file-name sanitizer, `.desktop` parser, icon re-encoder (malformed and oversized inputs), USB classifier (fake sysfs trees in `tmp_path`), gateway loop detection, fw_cfg schema, TOML writer round-trip. Hypothesis property tests for every function that handles guest-supplied data.
2. **libvirt** (`tests/libvirt`, `test:///default` driver, no KVM): define/start/stop, metadata store, networks, pools, dependency order, transient domains.
3. **End-to-end** (`tests/e2e`, `pytest -m e2e`, real `qemu:///system`, opt-in): template build; app over vsock with border/title; no capture source (guest `pactl list sources short` shows only monitors, no host capture stream); kill switch (gateway tunnel down → downstream `curl` fails); DNS only via the tunnel; VM↔VM ping blocked; VM→host ports closed except DNS; disposable leaves no disk files; USB device with an audio interface is refused; WireGuard config with `PostUp` is rejected.

## 14. Phases

Each phase gets its own implementation plan. The manager grows with each phase.

### Phase 0: Spikes (throwaway code; output is `docs/superpowers/specs/2026-10-04-spike-findings.md`, then an update of this spec)
| ID | Question | Success criterion | Fallback |
|---|---|---|---|
| S1 | Does xpra (Fedora 44) attach over native vsock, with border, title, and speaker working and mic off, under GNOME Wayland/XWayland? | App window shows with border and prefix; audio plays; no capture stream on host | SSH over vsock (§9.1) |
| S2 | With host Mullvad connected: does `vmp-uplink` NAT traffic work, and does a WireGuard handshake from a VM succeed? | Clear yes/no for both, plus the required Mullvad settings | §7.5 (a)/(b) |
| S3 | Does `<transient shareBacking='yes'/>` let two VMs share one template root, and are overlays cleaned up after shutdown and after `kill -9` of QEMU? | Both VMs run; no leftover overlays (or a defined cleanup) | Managed per-boot overlays (§6.2) |
| S4 | Whonix-Gateway KVM import: signature verification, boot with vmp XML, a Fedora client reaching Tor, external addressing, reconfigurability for chaining, console access (serial or graphics) | Fedora client gets a Tor exit IP; a console works | Chaining disabled (§7.4); graphics console for sys-tor only (§8) |
| S5 | From a GNOME-shortcut-launched process: find the focused xpra window's VM; read/write the host clipboard | Correct VM found; clipboard round-trip works | GTK chooser; `wl-clipboard` |
| S6 | usbguard + libvirt USB passthrough: blocked device, allowed device, D-Bus temporary allow | Defined, working attach flow | Doctor hint: allow in usbguard first |

### Phase 1: Foundation
Project skeleton, `install.sh`, setup helper, `setup`, `doctor`, `core` (names, model, store, conn, domain_xml, storage, images, templates, fwcfg, lifecycle, display, apps), guest units, `none`/`direct` networking with isolation and the `vmp` zone, vsock display + audio, `run`, `apps sync/expose`. Manager v0: VM list, start/stop, launch, doctor. **Result:** isolated app VMs whose display is unaffected by Mullvad. After Phase 1: tag the last Rust commit `rust-final` and remove `src/**/*.rs`, `Cargo.toml`, `Cargo.lock`, `tests/*.rs`, `test-xpra.sh` in one commit; rewrite README.

### Phase 2: Data flow
Disposables, clipboard push/pull + shortcuts, file copy, approval and chooser dialogs, open-in-disposable; matching manager actions.

### Phase 3: Gateways
`sys-vpn` (create, import, kill switch, DNS), `sys-tor` (Whonix import), chaining, `update_network`; manager network settings and gateway health.

### Phase 4: USB
Listing, allowlists, class rules, attach/detach, usbguard integration; manager USB menu.

## 15. Dropped from the Rust version

| Feature | Reason |
|---|---|
| DistroForge integration / Rust library API | Out of scope per user |
| GPU/PCI passthrough, CPU pinning, UEFI-for-GPU flow | Not kept; separate use case |
| virtiofs shared folders | Not kept; file copy replaces it |
| Selkies web streaming | Exposes VMs on the network |
| Bridged LAN networking | Replaced by the gateway model |
| SSH transport, `sshpass`, `StrictHostKeyChecking=no` | Replaced by vsock |
| Per-VM kickstart installs, custom kickstart injection | Replaced by verified cloud-image templates |
| Raw `iptables` rule strings | Replaced by fixed, generated nftables rules in gateways |
| `vm-passwords.toml`, guest passwords, NOPASSWD sudo, SELinux permissive | No passwords; no guest sudo; enforcing |
| Host PulseAudio socket forwarding | Replaced by xpra speaker forwarding (no mic path) |

## 16. Glossary
- **CID:** vsock context ID, a per-VM address on the host↔guest vsock channel.
- **qemu-guest-agent (QGA):** in-guest daemon on a virtio-serial channel; the host can run commands and read/write files through it. Only the host can start a request.
- **fw_cfg:** QEMU firmware config interface; libvirt passes named blobs that the guest reads from sysfs.
- **Transient disk:** a libvirt disk whose writes go to a temporary overlay that is deleted when the VM stops.
