# Bobcat 300 Repurpose Project
## Goal: MeshCore LoRa Repeater Node

---

## Device Identity

| Field | Value |
|-------|-------|
| Model | Bobcat Miner 300 |
| Frequency | US 915 MHz (compatible with AU915) |
| Power | DC 12V 1A |
| MAC1 (Ethernet) | E8:78:29:54:DD:30 |
| MAC2 (WiFi) | 48:E7:DA:B5:A7:B9 |
| FCC ID | 2AZCKMINER300 |
| Serial | 285US915M221301351 |
| Manufacturer | Chengdu Just Do It Information and Technology Co., Ltd |

---

## Hardware Components

### Main SoC
- **Rockchip**, RK356x family confirmed via FCC internal photos (Rev 2 board, silkscreen "G285-V1.0")
- Exact variant (RK3566 vs RK3568) still unconfirmed — FCC photo resolution insufficient to read final digit
- Paired with a "Rockchip RK809-x" PMIC
- Runs Linux
- Target platform: **RDK (Rockchip Development Kit)**

#### FCC filing note: two hardware revisions exist under FCC ID 2AZCKMINER300
- **Rev 1** (G280-V1.1, filed 2021-04-02): SoC = **Rockchip PX30** (quad-core Cortex-A35), LoRa = SX1308 + 2×SX1257 (8-channel), NOT SX1302/1303. Different device generation — do not confuse with our board.
- **Rev 2** (G285-V1.0, filed 2021-08-04): matches our board ("PCB-N350" LoRa module marking). SoC = RK356x (exact variant TBC as above).
- No schematic recoverable from either filing (listed but never uploaded, not confidentiality-redacted).
- Confirmed frequency bands (Rev 2 grant): 902.3–926 MHz (US915) + 2.4 GHz WiFi/BT only — no AS923/AU-specific band in this filing.

### WiFi/BT Module
- **AzureWave AW-NM372NM**
- WiFi + Bluetooth combo
- Handles local network connectivity
- RDK driver support TBC

### LoRa Concentrator
- **N3504S** (PCB-N350)
- Serial: MN3504S221303653
- Based on **Semtech SX1302/SX1303** chipset
- Multi-channel LoRaWAN concentrator
- Connected to Rockchip via a **mini-PCIe-style gold-finger edge connector** (confirmed via FCC internal photos), not loose SPI wires — pinout is Bobcat-custom, not standard mPCIe, and still TBC
- Open source HAL available: `github.com/Lora-net/sx1302_hal`
- U.FL antenna connector → coax pigtail → external SMA
- **Confirmed working under stock firmware**: a packet-forwarder process is running and connecting to `gateway_rs` locally (see Live Device Status below) — SX1302 HAL integration (Radio Integration Plan Step 1) is already solved by the vendor, don't need to redo it from scratch

### Storage
- eMMC (internal, contains stock Bobcat firmware)
- MicroSD slot (TF CARD) — can be used for alternate boot

---

## External Ports (Back Panel)

| Port | Type | Purpose |
|------|------|---------|
| LORA | SMA | LoRa antenna |
| BT BUTTON | Button | Bluetooth pairing/reset |
| TF CARD | MicroSD | Alternate boot / storage |
| COM | Micro USB | Serial console (UART) |
| ETHERNET | RJ45 | Network / SSH access |
| DC 12V | Barrel jack | Power input |

## Internal Ports (On Board)

| Port | Type | Purpose |
|------|------|---------|
| UART pads | 3x copper pads (TX/RX/GND) | Debug header (unpopulated) |
| USB-C | USB-C connector | Likely OTG/Maskrom flashing port |

---

## Live Device Status (as of 2026-07-28, first power-on)

- Device powers on and boots fully under stock firmware — no COM-port serial console needed for basic access
- **External COM port (micro-USB) did not enumerate as a USB device on the host** even with 12V applied and a known-good data cable — likely wired to Rockchip USB OTG (software-gadget serial), not a dedicated UART bridge chip. Not pursued further once network access was confirmed working.
- Device joined the LAN over **WiFi** automatically (already had credentials saved from prior setup) — MAC `48:e7:da:b5:a7:b9` matches documented MAC2, IP `192.168.4.17` (may change — check DHCP lease/router if reconnecting later)
- **Web dashboard** at `http://<device-ip>/` — stock "Bobcatminer Diagnostic Dashboard", firmware `v1.0.3.18`, hotspot animal name "energetic-shamrock-goldfish"
  - Login: `bobcat` / `miner` (HTTP Basic Auth) — works for `/log/console.log`; endpoints `/miner.json` and `/speed.json` are open without auth but rate-limited
  - `/miner.json` confirms: `region: us915`, miner runs as Docker container `quay.io/team-helium/miner:gateway-latest` ("Up 21 hours" at time of check) — **host OS is a general Linux system running Docker, not a monolithic firmware blob**
  - `/log/console.log` shows `gateway_rs` (Helium's Rust gateway, v1.3.0) listening on `0.0.0.0:1680`, receiving packets from a **local packet-forwarder client** on `127.0.0.1:33832` — confirms the SX1302 HAL/packet-forwarder is already running and working against the N3504S concentrator under stock firmware
- **SSH (port 22) was locked to public-key auth only** on first boot — `bobcat`/`miner` (dashboard creds) rejected with `Permission denied (publickey)`. Resolved via SD-card patch, see below.

### Root access obtained (2026-07-28, via SD-card patch — Method A)
Flashed `sd-image/patched/Bobcat_SSH_285_patched.img.xz` to a 64GB microSD (via USB reader, `/dev/sda`, confirmed by size/removable-flag/USB-topology match before writing), booted once with the card inserted, then removed it and power-cycled back to stock firmware. Confirmed working:

- `ssh admin@192.168.4.17` password `bobcat` — **succeeds**
- **`admin` has UID 0 / GID 0 — it IS root**, not a limited account
- Kernel: `Linux bobcatminer 4.19.172 #7 SMP Tue Nov 23 15:31:49 CST 2021 aarch64` — vendor BSP kernel, not mainline
- Root filesystem: `/dev/mmcblk0p4 on / type ext4 (rw,noatime,nodiratime)` — **writable**, not a read-only overlay
- **Retroactive validation of the `easylinkin` finding**: `/etc/passwd` on the real device already contains `easylinkin:x:1000:1000::/home/easylinkin:/sbin/nologin`. Our assumption that the account was "likely inert" (because we guessed stock `passwd` wouldn't define it) was **wrong** — it does exist. `/sbin/nologin` would block a full interactive shell, but SSH auth would still have succeeded for it had we not stripped it from the shadow file before flashing — confirms that was a real risk, not a hypothetical one.
- **Password left as `bobcat` for now, deliberately** — not yet changed. This is a known, temporary, LAN-only state; revisit before considering this device "done" (don't expose port 22 beyond LAN in the meantime, per the existing warning above).

### Pre-existing unknown SSH key found (significant — not something we installed)
`~/.ssh/authorized_keys` for `admin` already contained a 4096-bit RSA key **before we touched anything today** (file timestamp `Jul 28 07:01`, hours before this session started). Fingerprint: `SHA256:pV9x9AURR82/M13+T5uWpxumt0YNdXwlEPGCRiSFzO8`, no comment. This isn't from the sicXnull image (its payload only ever touches `shadow`/`shadow-`/`sshd_config`, never `authorized_keys`). Someone — Bobcat/manufacturer remote-support tooling, or a previous owner if this unit was acquired used — already had passwordless root access to this device. Left the key in place (didn't remove it, just preserved it while fixing an unrelated formatting issue — see below). Worth deciding later whether to strip it once we're confident we understand what it's for.

### SSH key set up for our own use, plus a config lesson learned
Added our own key (`~/.ssh/bobcat_lora_ed25519` on the work laptop) to `admin`'s `authorized_keys` for reliable automation. Two mistakes made and fixed along the way, worth remembering:
- A naive `echo >> authorized_keys` appended without a leading newline (the file didn't end in one), merging the pre-existing RSA key and our new key into one unparsable line. Fixed by rewriting the file with both keys on separate lines, preserving the original.
- `AuthenticationMethods publickey,password` (comma) means **require both, in sequence** (2FA) — not "either is fine." Needed `AuthenticationMethods publickey password` (space-separated) for OR semantics. Got this wrong once, which briefly required satisfying both factors in the same connection to fix. `sshd` here runs as a bare process (PID from boot, no systemd/supervisor) — reload config changes with `kill -HUP <sshd-master-pid>`, not a service restart, since there's nothing to respawn it if killed outright.
- Final working state: **both key-based (`~/.ssh/bobcat_lora_ed25519`) and password (`bobcat`) login work independently**, either is sufficient.

---

## System Exploration (root shell, 2026-07-28) — findings toward the MeshCore repeater goal

### Hardware, confirmed definitively
- **SoC: Rockchip RK3566** (was ambiguous between RK3566/RK3568 from FCC photos alone) — `/proc/device-tree/model`: "Rockchip RK3566 Tiansizhihui Board", compatible string `rockchip,rk3566-evb2-lp4x-v10 rockchip,rk3566`. The `evb2-lp4x` string suggests Bobcat's device tree started from Rockchip's own EVB2 LPDDR4x reference board — good news for any future "fresh Linux" fallback, since a public reference DT baseline likely exists for core peripherals (eMMC, UART) even though LoRa/WiFi peripherals were customized on top.
- RAM: ~2GB
- **WiFi/BT chipset confirmed as Broadcom BCM43438** (via `brcm_patchram_plus1 ... bcm43438a1.hcd /dev/ttyS1` and `dhd_*` kernel threads) — AzureWave AW-NM372NM is just the module packaging; the silicon inside is Broadcom, not some AzureWave-proprietary chip. WiFi uses Broadcom's `dhd` driver; BT patchram loads over `/dev/ttyS1` (1.5Mbaud implied by matching UART setup).
- **Debug UART correction**: the system console (`getty`) runs on **`ttyFIQ0` at 1500000 baud**, not 115200 as originally guessed in this doc. If the internal 3-pin UART debug header (TX/RX/GND) ever gets used, try 1.5Mbps first — this is Rockchip's standard debug-UART rate, not the general-purpose 115200 default.
- OS: **Yocto-based**, `ID="SKYSI"`, `VERSION="1.0 (sumo)"` — "sumo" = Yocto Project 2.5 (2018) release train. Custom ODM/vendor distro built on Yocto, not a generic Debian-like image (that description applies to the SD-card *unlock tool's own live-boot environment*, not the actual device OS — worth not conflating the two).
- Kernel: `4.19.172` aarch64, vendor BSP branch (not mainline).

### eMMC partition layout (confirmed, differs slightly from the SD-tool's assumption)
| Partition | Size | Mount/purpose |
|---|---|---|
| `mmcblk0p1` | 4MB | boot/idbloader |
| `mmcblk0p2` | 512MB | kernel |
| `mmcblk0p3` | 4GB | `/update` — OTA staging area, holds the LoRa packet-forwarder config too (see below), nearly empty |
| `mmcblk0p4` | ~53GB | **root filesystem AND `/var/lib/docker`** — same partition serves both |

`/dev/loop1` (which briefly looked like it might be a separate ephemeral rootfs image when `df` was checked mid-exploration) is confirmed via `losetup -a` to be a **whole-device loop wrapper around `mmcblk0p4` itself** (Docker's overlay2 driver does this for filesystem feature compatibility), not a separate file-backed image — so `/` and Docker state are both genuinely persistent on real eMMC, not RAM/ephemeral. **47GB free** on that partition — plenty of room for MeshCore build tooling.

### Docker environment
- Docker CE **20.10.10**, arm64, plain vanilla (not a fleet-management wrapper like Balena) — standard `docker` CLI applies directly.
- One container running: `miner` (image `quay.io/team-helium/miner:gateway-latest`, 10.3MB, just the `helium_gateway` Rust binary — no bundled packet-forwarder).
- Container is **`Privileged: true`** with **`NetworkMode: host`** — explicit `Devices` list only shows `/dev/i2c-1` (used for the Helium secure-element keypair chip, `GW_KEYPAIR=ecc://i2c-1:96?slot=0` — a Microchip ECC608-style crypto chip, unrelated to the radio), but privileged mode means it can actually reach any host device regardless of that list.
- **This means an official MeshCore-in-Docker approach is very plausible**: `docker run --privileged --network host` with access to `/dev/spidev1.0` and `/dev/gpiochip*` (see below) is exactly the same access pattern already proven to work here for the existing miner container.

### LoRa concentrator interface — fully confirmed, this is the key piece for radio integration
- Actual radio driver process runs **directly on the host**, not in a container: `lora_pkt_fwd_1302 -c /update/cfg/global_conf_1302.json` (binary at `/usr/bin/lora_pkt_fwd_1302`, 273KB)
- **SPI device: `/dev/spidev1.0`** (confirmed, single SPI device node present; kernel `[spi1]` thread active)
- **Radio chips: SX1302 baseband + 2× SX1250 RF front-ends** (not SX1257 as seen on the older Rev1/G280 hardware) — resolves the SX1302-vs-SX1303 ambiguity from the FCC-photo research: it's SX1302.
- Config (`/update/cfg/global_conf_1302.json`, persistent on `/update`) has the full US915 8-channel sub-band already tuned and working: channels 903.9–905.3MHz (multi-SF) + one 500kHz/SF8 "fat" channel at 904.6MHz, `radio_0` centered 904.3MHz (TX-capable), `radio_1` centered 905.0MHz (RX-only). `gateway_ID` is derived directly from the Ethernet MAC: `AAe8782954dd30BB` (EUI64-padded).
- Talks to `gateway_rs` via `server_address: 127.0.0.1`, ports 1680 up/down — matches what we already saw in the console log.
- `/update/cfg/` also has `global_conf.json` (5.4KB, likely Semtech reference defaults before site-specific override) and `local_conf.json` (291 bytes, just overrides `gateway_ID`) — worth diffing against `global_conf_1302.json` later if we need to understand the full Semtech HAL config schema for our own use.
- No explicit GPIO reset pin found in this config — SX1302 reset handling may be inside the binary itself or a wrapper script not yet located. Not a blocker, just an open item.

### Region / frequency plan — still needs changing to AU915 before deployment
Currently live config is **US915** (`GW_REGION=US915` in the container env, and `global_conf_1302.json`'s channel plan is the standard 8-channel US915 sub-band centered on `radio_0`/`radio_1` at 904.3/905.0MHz). This is purely declarative JSON, read at `lora_pkt_fwd_1302` startup — **switching region is a config edit, not a firmware or code change**, and doesn't depend on whichever MeshCore radio-integration path we end up choosing (see "Docker container vs host process" discussion above) — both paths will need this same regional config addressed one way or another.

- `radio_0.tx_freq_min`/`tx_freq_max` in the current config is `902000000`–`928000000` — AU915's uplink band (~915.2–927.8MHz) falls **inside** that already-configured tunable range, so the SX1250 front-end's PLL can reach AU915 frequencies as configured today.
- **Unconfirmed**: whether the board's analog matching network/RF filtering was tuned broadly across that whole 902–928MHz range, or narrowly optimized for the US915 sub-band actually in use. The JSON can't answer this — would need either RF measurement or a schematic (none available, see FCC filing findings above) to be sure. Worth testing empirically (TX power/RX sensitivity at AU915-specific frequencies) once we're actually driving the radio ourselves, rather than assuming it'll be fine.
- Actual AU915 channel-plan values (which 8 channels, which frequencies) still need sourcing — not yet looked up.
- Revisit this once MeshCore's own radio/HAL code is understood, since it determines whether this JSON is still the relevant config surface or gets replaced by something else entirely.

### Other host processes worth knowing about
- `connmand` + `wpa_supplicant` — ConnMan-based WiFi management (not NetworkManager/systemd-networkd)
- `bluetoothd` + `brcm_patchram_plus1` — standard BlueZ stack, BCM43438 firmware loaded via `/dev/ttyS1`
- `gateway_config` — a separate **Erlang/OTP application** (`/opt/gateway_config/`) handling onboarding/provisioning; reads GPIO line 6 as input via `erlang-ale` (purpose not yet investigated — possibly a physical button or reset-detection line)
- `ota_daemon_new` — OTA update daemon (explains the `/update` partition's purpose)
- `diagnoser` — the web dashboard we've been using since early in this project
- `ubusd` — OpenWRT-style micro-bus daemon present as a standalone component (system isn't full OpenWRT, just borrows this)

### Implications for the MeshCore plan
1. **Radio integration Step 1 ("get SX1302 HAL running") is moot — it's already running and proven working** on this exact hardware, at `/dev/spidev1.0`, with a known-good config as a reference.
2. **Filesystem is genuinely persistent and has 47GB free** — plenty of room, no read-only-overlay obstacle.
3. **Docker is the natural install path** — same `--privileged --network host` pattern the stock miner container already uses would give a MeshCore container the same device access.
4. **The existing `lora_pkt_fwd_1302` process holds the SPI bus exclusively** — it will need to be stopped (not necessarily uninstalled — keep it as a fallback/reference) before anything else can claim the concentrator, since only one process can hold that interface at a time.

### Miner container stopped (2026-07-28)
Ran `docker stop miner`. Its restart policy is `unless-stopped` (confirmed via earlier `docker inspect`), which Docker honors across daemon/host restarts — a manual stop should persist through reboot without needing any init-script changes. **Not yet verified against an actual reboot** — worth confirming next time the device is power-cycled.

Important: **this does not free the radio.** `lora_pkt_fwd_1302` (the actual SPI-bus holder) is a separate host-level process, not started or managed by Docker, and is unaffected by this — it keeps demodulating on the US915 channel plan and forwarding to a now-dead UDP port (packets just silently drop, no functional harm). Stopping it too, to actually free `/dev/spidev1.0` for our own use, is a separate decision not yet made — need to find how it's started at boot (not Docker-managed, likely a crond entry or init script) before touching it.

**Decision (2026-07-28): keep `lora_pkt_fwd_1302` running.** Not stopping/disabling it — no action needed here.

---

## MeshCore codebase research (2026-07-28) — significant architectural mismatch found

Cloned `https://github.com/meshcore-dev/meshcore` (MIT license, C/C++, 3.3k stars) for exploration, toward the goal of a Docker-deployable MeshCore repeater for this hardware, forking if needed.

### What MeshCore actually is
- **Microcontroller firmware**, built via PlatformIO, targeting ESP32-S3/nRF52840/STM32/RP2040 boards (Heltec, RAK, LilyGo, etc. — 79 board variants in `variants/`). Flashed as a compiled binary/UF2, not run as a Linux service.
- Radio access goes entirely through **RadioLib** (`jgromes/RadioLib`), which drives **single-chip LoRa transceivers directly over SPI** — confirmed wrapper support exists only for **SX1276, SX1268, SX1262** (`src/helpers/radiolib/Custom{SX1276,SX1268,SX1262}*`). No SX1302/1303 support anywhere.
- Core routing/protocol logic (`src/Mesh.cpp`, `Dispatcher.cpp`, `Packet.cpp`, `Identity.cpp`) is reasonably hardware-agnostic C++, but `examples/simple_repeater/main.cpp` is deeply Arduino-framework-coupled: `#include <Arduino.h>`, `Serial.begin()`, and platform-specific filesystem branches (`InternalFS`/`SPIFFS`/`LittleFS`) with `#error "need to define filesystem"` if none match — **no Linux branch exists**.
- There IS a `[env:native]` PlatformIO environment, but it's **unit-test-only** (hardware fully mocked via `test/mocks/{Stream,SHA256,AES}.h`), not a deployable build — confirmed via the CI workflow, which only runs `pio test -e native`.
- **Board porting pattern** (relevant if we go the port route): each variant is `platformio.ini` + `target.h`/`target.cpp` (defines the `board`, `radio_driver`, `radio_init()` globals + pin mappings) + a `*Board.h` subclass. Well-templated for adding new microcontroller+RadioLib-chip combos — not templated at all for a Linux+concentrator combo, that would be new ground.

### The mismatch, plainly
Our hardware has a **multi-channel SX1302 concentrator behind a UDP packet-forwarder protocol**, driven by a full Linux userspace on an RK3566 SoC. MeshCore expects **direct SPI ownership of a single-chip transceiver from microcontroller firmware**. These are two different ecosystems (RadioLib vs. Semtech's `sx1302_hal`) that don't currently talk to each other anywhere in the MeshCore codebase.

### Three real paths forward
1. **Attach an external SX1262 module** (SPI or USB breakout) to the RK3566, bypass the SX1302 concentrator entirely, and get MeshCore running via RadioLib against that single chip. This was already the documented fallback in this doc's original Radio Integration Plan. RadioLib itself isn't Arduino-only — it accepts a custom `Hal` implementation for non-Arduino platforms (precedent exists in the RadioLib community for Raspberry Pi–style Linux SPI/GPIO), so this is "port RadioLib's Hal + MeshCore's Arduino-coupled bits to Linux," not "invent a new radio driver." Smallest, most bounded scope; costs one small piece of extra hardware (~US$10-20) per device; most realistically repeatable for other people repurposing their own Bobcats since the software side stays close to upstream.
2. **Bridge MeshCore to the existing `lora_pkt_fwd_1302` process** via its already-working UDP packet-forwarder protocol on `127.0.0.1:1680`, writing a new "radio driver" that receives/sends demodulated packets over UDP instead of raw SPI. No new hardware needed, reuses the proven-working concentrator config as-is. Requires writing a real MeshCore-side radio backend from scratch (nothing to adapt from upstream, since RadioLib has no UDP/packet-forwarder concept) and reconciling MeshCore's expected single-channel behavior against the concentrator's inherently multi-channel/LoRaWAN-shaped packet metadata.
3. **Full native SX1302 HAL integration** — write a genuinely new low-level MeshCore radio backend using Semtech's `sx1302_hal` directly, taking exclusive control of `/dev/spidev1.0` ourselves (would require stopping `lora_pkt_fwd_1302`, reversing the "keep it running" decision above). Most "correct"/native long-term, but the largest and most unprecedented scope of the three — nothing in either codebase to build from.

### Recommendation, pending discussion
Path 1 (external SX1262 module) is the smallest bounded effort, closest to how upstream MeshCore already works elsewhere (lowest ongoing maintenance burden against upstream changes), and most reproducible for the stated goal of helping others repurpose their own Bobcat 300s — "buy a $15 module, run our image" is a much easier thing to hand to someone else than "carry a from-scratch radio driver port." Paths 2 and 3 avoid new hardware but commit to real, unprecedented driver-level engineering with no upstream template to follow.

**Decision (2026-07-28): Path 2 — bridge to the existing `lora_pkt_fwd_1302` via its UDP protocol.** No new hardware. Details below.

### Path 2 implementation plan — interface mapping

**MeshCore side — `mesh::Radio` abstract interface** (`src/Dispatcher.h`), the actual contract to implement. Much cleaner than expected — this is what `RadioLibWrapper` itself implements, so we don't need to touch RadioLib or fake being a `PhysicalLayer` at all, just implement this directly:
```cpp
class Radio {
  virtual void begin();
  virtual int recvRaw(uint8_t* bytes, int sz) = 0;         // poll for next received packet
  virtual uint32_t getEstAirtimeFor(int len_bytes) = 0;     // standard LoRa airtime formula
  virtual float packetScore(float snr, int packet_len) = 0; // routing heuristic, can copy RadioLibWrapper's packetScoreInt
  virtual bool startSendRaw(const uint8_t* bytes, int len) = 0;
  virtual bool isSendComplete() = 0;
  virtual void onSendFinished();
  virtual void loop();          // do periodic work here
  virtual bool isInRecvMode() const = 0;
  virtual bool isReceiving();
  virtual float getLastRSSI() const;
  virtual float getLastSNR() const;
};
```

**`lora_pkt_fwd_1302` side — Semtech UDP packet-forwarder protocol** (spec: `Lora-net/packet_forwarder` `PROTOCOL.TXT`, cloned for reference). Binary header + JSON payload over UDP:

| Message | Direction | Purpose |
|---|---|---|
| `PUSH_DATA` (0x00) | gateway→server | Uplink: JSON `rxpk` array (one object per received packet: `freq`, `chan`, `rfch`, `datr` e.g. `"SF8BW500"`, `codr`, `rssi`, `lsnr`, `size`, `data` base64) — this is where our received-packet metadata and payload come from |
| `PUSH_ACK` (0x01) | server→gateway | Must ack every `PUSH_DATA` immediately |
| `PULL_DATA` (0x02) | gateway→server | Keepalive, sent periodically by the gateway — **also the only way to learn the gateway's current source address/port for sending it downlink**, since it may be behind NAT |
| `PULL_ACK` (0x04) | server→gateway | Ack of `PULL_DATA`; confirms the route is open for downlink |
| `PULL_RESP` (0x03) | server→gateway | Downstream: JSON `txpk` object (`freq`, `rfch`, `powe`, `modu:"LORA"`, `datr`, `codr`, `size`, `data` base64, `imme:true` for immediate send) — this is how we transmit |
| `TX_ACK` (0x05) | gateway→server | Gateway's accept/reject feedback after a `PULL_RESP` |

Since **we already stopped the `miner` container** (freeing port 1680, previously held by `gateway_rs`), our own process can bind `127.0.0.1:1680` and act as the "server" role `lora_pkt_fwd_1302` is already configured to talk to (`server_address: 127.0.0.1`, `serv_port_up`/`serv_port_down: 1680` in `global_conf_1302.json`) — **no packet-forwarder config changes needed**, it's already pointed at exactly the address/port we'll listen on.

**Interface mapping (concrete, not yet implemented):**
- `recvRaw` ← pull next decoded payload from an internal queue populated by parsing incoming `PUSH_DATA` JSON (`rxpk[].data`, base64-decoded)
- `startSendRaw` → build a `txpk` JSON object, wrap in a `PULL_RESP` binary header with a random token, send via UDP to whichever address/port the last `PULL_DATA` came from
- `getLastRSSI`/`getLastSNR` ← directly from the most recent `rxpk[].rssi`/`.lsnr` fields, no computation needed
- `getEstAirtimeFor` — standard LoRa airtime formula; needs one fixed SF/BW/CR chosen from the 8 available `chan_multiSF_*` channels or the `chan_Lora_std` "fat" channel (904.6MHz/500kHz BW/SF8) in the existing concentrator config, since MeshCore expects single-channel semantics
- `loop()` — non-blocking poll of the UDP socket, JSON parsing, periodic `PULL_ACK` bookkeeping

**Open items not yet resolved:**
- Which specific channel/SF to fix MeshCore's logical single channel to, from the 8 already configured in `global_conf_1302.json`
- JSON parsing approach — MeshCore doesn't bundle a JSON library; likely need to add one (e.g. ArduinoJson) or hand-roll parsing given the fairly fixed schema
- Linux/Arduino-framework compatibility shims still needed regardless of radio backend (`Serial`, `millis()`, filesystem — see architectural mismatch notes above) — this work is required either way, not specific to Path 2
- Fork location/name not yet decided
- Docker base image / arm64 cross-compile toolchain not yet decided

### MeshCore Linux/UDP-bridge implementation — Phase 0 complete (2026-07-28)
Fork: `https://github.com/jezzam/MeshCore` (added as git submodule at `meshcore/` in this repo, `origin` = fork, `upstream` = `meshcore-dev/meshcore`, for easy syncing). New variant: `meshcore/variants/linux_udp_bridge/`. Full design in the approved plan (`~/.claude/plans/whimsical-snuggling-dragon.md`).

**Phase 0 (compile + link + smoke-test run, stub radio) is done and verified working** — not just compiling, but a real end-to-end smoke test: Ed25519 identity generation, hex-encoding, file persistence (`IdentityStore`), and the admin CLI loop all ran correctly against real vendored crypto (libsodium for SHA256/HMAC/Ed25519-verify, vendored tiny-AES-c for AES-128-ECB), with zero infinite-loop/CPU-spin bugs after one found-and-fixed issue (see below).

Built via the actual committed `variants/linux_udp_bridge/CMakeLists.txt` (not just ad-hoc test compiles) — confirmed with libsodium-dev headers extracted locally without root (`apt-get download` + `dpkg-deb -x`, no `sudo` needed) and a standalone `cmake` binary installed to `~/.local/bin`. Only 3 small upstream edits needed (`main.cpp`, `MyMesh.cpp` x2, `IdentityStore.h` x1 one-liner) — everything else is new files under `variants/linux_udp_bridge/`, matching the plan's fork-stays-mergeable design.

**Real bug found and fixed via the smoke test** (a good example of why "trust but verify" matters even for AI-written shims): the `Serial`-over-stdin shim's `available()` didn't handle stdin EOF correctly — with stdin closed (the normal case for `docker run` without `-it`), `select()` reports a closed stdin as "readable" forever, causing an infinite busy-loop appending garbage bytes and printing "Unknown command" at 100% CPU forever. Fixed by tracking an EOF flag once `read()` returns 0.

**What's NOT yet real**: `PktFwdRadio` is currently a stub (always "nothing received", always "send failed") — Phase 1 (real UDP protocol handshake against the live `lora_pkt_fwd_1302`) is next, not yet started.

### Phase 1 complete (2026-07-28) — real UDP bridge, validated against a simulated gateway
`PktFwdRadio` (`variants/linux_udp_bridge/PktFwdRadio.{h,cpp}`) now fully implements the Semtech UDP packet-forwarder protocol: binds `127.0.0.1:1680`, handles `PUSH_DATA`/`PULL_DATA` with immediate acks, tracks the downlink address from `PULL_DATA` (the only way to learn it, per spec), sends `PULL_RESP` for TX with `imme:true`, real LoRa airtime formula (AN1200.13) against our fixed channel (904.6MHz/500kHz BW/SF8, matching `chan_Lora_std` in `global_conf_1302.json`), and `packetScore` ported from `RadioLibWrapper::packetScoreInt`. JSON via vendored `nlohmann/json` single header; base64 via libsodium (already a dependency).

**Validated with a Python script (`fake_gateway.py`, not committed — recreate on demand) simulating the gateway side of the protocol, not yet against the real device.** All three tests passed:
1. `PULL_DATA` → `PULL_ACK`
2. `PUSH_DATA` (fake rxpk) → `PUSH_ACK`
3. **The real 16-second boot-time self-advertisement fired automatically and arrived as a correctly-formed `PULL_RESP`** — decoded payload contains the actual generated Ed25519 public key and the ASCII "repeater" role string. This is a real, correctly-encoded MeshCore packet, not just a protocol-level ack — meaningfully further validated than "Phase 1" originally scoped (no mesh logic, just log received packets).

**A real bug found and fixed along the way**: `Dispatcher::begin()` (upstream, unmodified) calls `_radio->begin()` as part of its own normal startup sequence, *in addition to* our own explicit call from `radio_init()` — harmless for real radio hardware (idempotent by nature), but our first implementation unconditionally recreated and rebound the UDP socket on every call, so the second call silently created an orphaned, never-bound socket (the bind failed with "address in use" against our own first socket, but the failure was invisible — `MESH_DEBUG_PRINTLN` is a no-op without `-DMESH_DEBUG`, which we don't define). `loop()` then read from the wrong fd forever, receiving nothing, no error, no crash — just silence. Root-caused by comparing the `this` pointer and file descriptor at both call sites via temporary debug tracing; found they were the same object with a changed `_sock` value, which pinpointed a second `begin()` call rather than an object-identity bug. Fixed by making `begin()` idempotent (no-op if already bound) and switching the failure path to `Serial.printf` (always visible) instead of the debug macro.

**Not yet done**: real on-device test against the actual `lora_pkt_fwd_1302` process (only tested against a local protocol simulator so far), and the Dockerfile/packaging.

### Deployed and validated on the real device (2026-07-28)
Repo split per the plan: fork (`meshcore/` submodule) stays generic; Bobcat-specific deployment packaging lives in `meshcore-deploy/` in this repo — `Dockerfile` (two-stage, `debian:bookworm-slim`, builds the `linux_udp_bridge` variant via the same `CMakeLists.txt`) and `docker-compose.yml` (`network_mode: host` — required, since `lora_pkt_fwd_1302` is configured for literal `server_address: 127.0.0.1`, and a container's loopback is its own network-namespace-local loopback, unreachable from the host's without host networking; **not** `--privileged`, since this container touches no device nodes at all). Added `.dockerignore` to keep the build context small (excludes the 250MB+ SD-image binaries and irrelevant MeshCore board variants/docs).

Transferred to the device via `rsync` over SSH (not `git push` — deliberately kept the fork's git history untouched pending explicit go-ahead to commit/push). Built **natively on the device's own arm64 CPU** inside Docker (no cross-compilation needed) — clean build on the very first attempt after fixing an `.dockerignore`-adjacent bug (rsync's include/exclude filter order was backwards, silently excluding the whole `variants/linux_udp_bridge/` directory — rsync's first-matching-rule-wins semantics require broad excludes to come *after* specific includes, not before).

One real infrastructure issue found and fixed: `docker build`'s default bridge network couldn't resolve `deb.debian.org` on this device (DNS failure), even though `--network host` containers (the actual runtime mode) have working internet access — fixed with `docker build --network host`, forcing the build-time `apt-get` calls onto host networking too.

**Confirmed working end-to-end against the real hardware**: container starts, binds `127.0.0.1:1680`, generates and persists a real Ed25519 identity to `/update/meshcore-data` (bind-mounted, survives container restarts), runs stably with no crashes. Verified via `/proc/net/udp` (cross-referenced against `/proc/1000/fd/` — PID 1000 is the real `lora_pkt_fwd_1302`) that **the real packet-forwarder process has two live connected UDP sockets talking to our container's port 1680** — genuine two-way protocol traffic with the actual concentrator hardware path, not a simulation.

**What's still unverified**: actual over-the-air RF reception/transmission and real mesh routing behavior with another MeshCore node. The `miner` container remains stopped as before.

### Retuned to the real local MeshCore channel — Brisbane/SEQ default (2026-07-28)
User is in Brisbane/SEQ, with existing MeshCore repeaters nearby — real over-the-air testing is possible once channel-matched (MeshCore requires exact frequency/BW/SF agreement; being in range doesn't help otherwise). Researched Australian MeshCore presets (web search + the user pasting the actual Brisbane/SEQ community page content, since automated fetching of that page failed):

| Preset | Frequency | BW | SF | CR | Where |
|---|---|---|---|---|---|
| Australia (Mid) | 915.075 MHz | 125 kHz | 9 | 4/5 | Greater Sydney |
| Australia (Narrow) | 916.575 MHz | 62.5 kHz | 7 | 4/8 | EastMesh default, regional NSW/ACT |
| Australia (Wide) | 915.800 MHz | 250 kHz | 11 | 4/5 | Other states (per a GitHub issue correcting this preset's SF) |
| **Brisbane/SEQ (in use)** | **923.125 MHz** | **62.5 kHz** | **8** | **4/6** | wiki.mbug.com.au/en/Meshcore/Settings — **this is what we configured** |

Confirmed: these presets are **mutually exclusive** — devices must match exactly to talk directly to each other. CR is explicitly a recommendation, not fixed — intended to be lowered as the local mesh gains more direct neighbours.

**Made runtime-configurable** (user's call, and a good one): `LORA_FREQ/BW/SF/CR` CMake `-D` flags are now just fallback defaults; `PktFwdRadio::begin()` reads `MESHCORE_LORA_FREQ/BW/SF/CR` env vars at startup and overrides them, so switching presets or tuning CR down later doesn't need a rebuild — just restart the container with different `docker-compose.yml` environment values.

### Major discovery: `global_conf_1302.json` is NOT persistently editable directly
First attempt: edited `/update/cfg/global_conf_1302.json` directly (retuned `radio_0` to 923.125MHz, `chan_Lora_std` to 62.5kHz/SF8, disabled the now-irrelevant multi-SF channels), rebooted to test persistence. **The edit was completely gone after reboot** — the file came back byte-for-byte identical to our pre-edit backup.

Root cause, found by reading the actual boot scripts (`/etc/init.d/lora-sx1302.sh` → `/usr/local/bin/easylinkin_lora_cfg_update_1302.sh` → `/etc/lora_cfg_check_1302.sh`):
- `easylinkin_lora_cfg_update_1302.sh`'s `update_web_conf()` runs **unconditionally on every boot** and always does `cp /etc/lora_config/global_conf_sx1250_${CONF}.json → /update/cfg/global_conf_1302.json`, where `${CONF}` comes from UCI (`uci get lora.@rsconf[0].radio_conf`, config file `/etc/config/lora`) — **stock value was `us915`**, so every boot silently restamped the stock US915 template over any edit, then regenerated `gateway_ID` deterministically from the Ethernet MAC (explaining why the "reverted" file matched our backup exactly — same template + same deterministic MAC-derived ID, every time).
- Separately, `/etc/lora_cfg_check_1302.sh` validates the file by stripping comments with a **single-line-only** sed (`s/\/\*.*\*\///`) then `python json.loads()` — our first edit used multi-line `/* */` comments, which would have failed this validation too (triggering its own separate repair-from-`cn470`-template fallback), independent of the UCI issue above.

**Fix**: `easylinkin_lora_cfg_update_1302.sh` has a `set_costumize_freq_conf()` function, only invoked when `radio_conf` contains `"customize"`, that reads frequency/channel-offset overrides straight from UCI — clearly the vendor's intended customization path. Set `uci set lora.@rsconf[0].radio_conf='customize'; uci commit lora`. No `global_conf_sx1250_customize.json` template actually exists on this device, so the unconditional `cp` now fails harmlessly (source missing, destination untouched) rather than clobbering our edit — confirmed by dry-running both boot scripts manually (`md5sum` of the config file identical before and after both scripts ran). Re-applied the config edit with single-line comments only. **Verified surviving an actual reboot this time** — `lora_pkt_fwd_1302`'s own startup log confirms `radio 0 ... center frequency 923125000` and `Lora std channel> radio 0, IF 0 Hz, 62500 Hz bw, SF 8`.

Backup of the original stock config preserved at `/update/cfg/global_conf_1302.json.bak-pre-meshcore` on the device (not committed to this repo).

### Known-benign oddity: `lora_pkt_fwd_1302` runs as two processes
Every manual restart (three separate attempts, including via the proper `/etc/init.d/lora-sx1302.sh start`) produces **two** `lora_pkt_fwd_1302` processes, both with `/dev/spidev1.0` open, both consuming similar CPU. Investigated for a real SPI-contention bug (checked `dmesg` for errors, compared fds, checked parent PIDs) — found no evidence of actual harm: no kernel/SPI errors ever logged, the daemon's own stats show clean `PUSH_DATA`/`PULL_DATA` exchange with 100% ack rate, and the pattern is perfectly consistent across every restart, suggesting it's just an intrinsic (if unexplained) trait of this binary's process model rather than something our restarts are causing. Not investigated further given no observed functional impact — worth keeping an eye on if anything RF-related seems flaky later.

### Restart procedure reference (for next time)
Don't manually `nohup`/`setsid` the binary directly — use the real init script, which correctly threads through the UCI config check and validation steps:
```
PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin /etc/init.d/lora-sx1302.sh stop
PATH=/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/bin /etc/init.d/lora-sx1302.sh start
```
Note: `start` backgrounds the daemon with **no output redirection**, so it'll stream its live stats block forever into your SSH session if left in the foreground — always run with `< /dev/null` and either accept a short pause or expect to background/detach your own SSH call, or you'll hit your own tool's timeout waiting for a pipe that never closes.

---

## SSH / Root Access Research

### Method A: SD-card boot patches eMMC shadow/sshd_config (community-documented)
- Source: `https://github.com/sicXnull/Bobcat-SSH-Access`
- Prebuilt SD images for three hardware versions (by serial prefix): 280, **285**, 29x — **our device's serial `285US915M221301351` matches the "285" image**
- How it works: SD-boots a minimal OS whose `/sbin/init` mounts the eMMC, backs up `/etc/shadow` and `/etc/ssh/sshd_config`, overwrites them with modified versions enabling password auth, then sets them immutable (`chattr +i`) so OTA updates can't silently revert it
- After running once, remove SD card and reboot to stock firmware — SSH then accepts `admin`/`bobcat`
- **Warning from the repo author**: never expose port 22 externally with these credentials live — LAN-only
- Not fully non-destructive (modifies eMMC files directly, though it backs up originals first) — treat as a real firmware modification, not a read-only probe
- Related community thread (unconfirmed content — Reddit blocks automated fetching): `https://www.reddit.com/r/HeliumMiners/comments/upl5bq/bobcat_300_unlock_ssh_access/`

### Method B: Recovery-mode ADB filesystem access (likely patched on our firmware)
- Source: `https://cybertuz.com/blog/post/Unlock_SSH_Bobcat_300_Miner`
- Different board silkscreen ("TU-GM1002Z"), confirmed **RK3566** SoC, firmware **v1.0.2.91** — corroborates RK356x family but from an older/different board revision than ours
- Method: hold physical recovery button while applying power → device enumerates as Android ADB interface over USB-OTG → `adb shell` gives root as `admin` → mount rootfs partition (`mount /dev/block/by-name/rootfs /mnt/sdcard`) → replace `/home/admin/.ssh/authorized_keys` with own key, tighten `sshd_config` to key-only auth
- **Author explicitly states the exploit was patched in later firmware, and their own device was remotely patched mid-use** — our firmware (v1.0.3.18) is newer than the exploited version, so this is likely non-functional on our unit as-is
- Possible connection to existing hardware: the "recovery button" they describe may be the same physical button already documented as **BT BUTTON** on our back panel — untested hypothesis, holding it during power-on is non-destructive to try
- Security note from the article: their published SSH key appeared reused across multiple devices — not relevant to us directly (we'd generate our own key), but a reminder not to reuse any key/credential we see published in these writeups

### Recommendation
Try **Method A (SD-card, sicXnull repo)** first — it's the one confirmed to match our exact hardware serial prefix ("285"), doesn't depend on firmware version, and doesn't require testing an unconfirmed recovery-button hypothesis.

### Method A — verified findings from direct image inspection (2026-07-28)
Downloaded `Bobcat_SSH_285.img.xz` directly (GitHub release 1.0, 129,746,800 bytes, sha256 `135b70391cf2d617212af99d069a28b0a91382c76c10a018ce4f90ad16454a9a` — no official checksum published by the author to compare against). Repo contains only a README, no source — inspected the image contents directly instead of trusting the description alone.

- **Image layout**: GPT, 3 partitions — `uboot` (4M), `skyboot` (fat16, 52.4M), `update` (ext4, 410M, the actual live-boot rootfs)
- **Real boot script found at `/usr/sbin/init`** (a plain shell script, not real systemd): mounts `/dev/mmcblk0p4` (**confirms our device's eMMC rootfs partition**) rw at `/mnt`, clears immutable flag on existing `shadow`/`shadow-`/`sshd_config`, backs them up as `.bak` (renamed, not deleted — a revert path exists), copies `/sshfiles/*` over them, re-sets immutable flag, then blinks a GPIO LED forever. **No auto-reboot, no error checking** — if the mount/copy fails, it still reaches "done" and blinks the LED regardless, so LED blinking is not proof of success.
- **`sshd_config` confirmed**: `PermitRootLogin yes` + `PasswordAuthentication yes` — root gets password access too, not just `admin` as the README implies.
- **Confirmed via hash verification**: both `admin` and `root` passwords are literally `bobcat`.
- **Found and removed**: an undocumented third account, `easylinkin`, with an active (non-locked) password hash in the staged `shadow` and `shadow-` files — present in both, never mentioned in the README. Likely inert in practice since the script never touches `/etc/passwd` (so the username probably can't resolve to a UID on stock Bobcat firmware), but real enough that we stripped it from our copy before flashing rather than trust that assumption.
- **Patched, verified image**: `easylinkin` removed from both shadow files, `admin`/`root` left as documented (`bobcat` — change this yourself once you have access, don't expose port 22 beyond LAN). Verified byte-for-byte merge into the full disk image, GPT table intact, `fsck -fn` clean before flashing.

---

## Access Methods

### Option 1: COM Port (Micro USB) — Serial Console
- Plug Micro USB → PC
- Open serial terminal: **115200 baud, 8N1**
- Windows: PuTTY
- Mac/Linux: `screen /dev/tty.usbserial 115200`
- Check for new port:
  - Windows: Device Manager → Ports (COM & LPT)
  - Mac: `ls /dev/tty.*`
  - Linux: `ls /dev/ttyUSB*`
- **Cannot flash via this port** — console only
- If existing firmware gives root shell, could write to eMMC from within Linux

### Option 2: MicroSD Card Boot — Safest Option
- Write RDK image to SD card
- Insert into TF CARD slot
- Bootloader may check SD first → boots RDK without touching eMMC
- Fully reversible — remove card to revert
- **Recommended first approach**

### Option 3: Internal USB-C — Maskrom Flashing
- Requires case open (already done)
- Rockchip Maskrom mode = chip-level flashing
- Tool: `rkdeveloptool` on PC
- Flashes RDK directly to eMMC
- **Last resort — irreversible without reflash**

---

## Recommended Approach (In Order)

1. **Serial console first** — plug COM port, power on, read boot log
   - Confirms exact Rockchip variant
   - See what existing firmware provides
   - Explore before changing anything
2. **Try SD card boot** — write RDK image, insert, power on
3. **SSH in** via Ethernet once RDK is running
4. **Maskrom flash** only if SD boot fails

---

## Australian Legal Compliance

- AU915 MHz is **licence-exempt** under ACMA class licence
- The N3504S operates in this band — legal for use
- MeshCore uses standard LoRa modulation — compliant
- Device is **NOT an SDR** — it is fixed-function LoRa hardware
  - Cannot tune to arbitrary frequencies
  - Regulated as a narrowband LoRa device
- US 915 MHz band partially overlaps AU915 — verify firmware supports AU915 channel plan

---

## MeshCore Role: Repeater

Chosen because:
- Device is at an **elevated fixed location**
- Mains powered (12V from existing PSU)
- Goal is to **extend network range** for nearby clients
- Repeater passively forwards all packets — no user interaction needed
- Runs unattended 24/7

### Potential upgrade: Infrastructure Node
- Add Ethernet backhaul to internet
- Significantly more useful than RF-only repeater

---

## Radio Integration Plan: SX1302 → MeshCore

### Challenge
- N3504S is a **multi-channel LoRaWAN concentrator**
- MeshCore typically expects single-channel SX126x/SX127x radios
- Requires adaptation work

### Steps
1. Get **SX1302 HAL** running on Rockchip
   - Identify SPI bus connection between Rockchip and N3504S (pinout TBC)
   - Compile `packet_forwarder` from `github.com/Lora-net/sx1302_hal`
   - Test basic radio comms
2. Adapt **MeshCore Linux** to use SX1302 backend
   - MeshCore abstracts radio via driver interface
   - Write/adapt SX1302 driver for MeshCore
   - SX1302 can operate in single-channel mode to match MeshCore expectations
3. Configure for **AU915 channel plan**

### Fallback Option
If SX1302 integration proves too difficult:
- Attach external **SX1262 USB LoRa module** to Rockchip
- MeshCore natively supports SX1262
- Faster path to working node

---

## Outstanding Unknowns

- [ ] Exact Rockchip SoC variant — narrowed to RK356x family via FCC photos, RK3566 vs RK3568 still unconfirmed
- [ ] SPI pinout between Rockchip and N3504S — confirmed to route via a Bobcat-custom mini-PCIe-style edge connector (FCC photos), exact pin mapping still TBC; no schematic available
- [ ] Whether bootloader checks MicroSD before eMMC
- [ ] AzureWave WiFi driver support in RDK
- [x] ~~Whether existing Bobcat firmware exposes root shell via COM port~~ — COM port doesn't enumerate as USB at all; superseded by network access (dashboard confirmed working)
- [ ] How to get host-level (Docker/OS) shell access — SSH is public-key-only, no known key. Options: internal UART pads, Maskrom flash, or finding an exec/shell path via the Docker container itself
- [ ] What the internal USB-C connector actually connects to — not visible in any FCC internal-photo exhibit for either hardware revision
- [ ] Whether AU915 is selectable in firmware config, or only enforced via `region`/`frequency_plan` fields seen in `/miner.json` (currently `us915`)

---

## Useful Resources

- SX1302 HAL: `https://github.com/Lora-net/sx1302_hal`
- rkdeveloptool: `https://github.com/rockchip-linux/rkdeveloptool`
- MeshCore: `https://github.com/ripplebiz/MeshCore`
- FCC filing (internal photos, RF specs): search FCC ID `2AZCKMINER300` at fcc.gov
- ACMA class licence (AU915): `https://www.acma.gov.au`