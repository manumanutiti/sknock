# Sknock — Vision

## When would I be satisfied enough to stop?

When any device — a soil sensor in a remote farm, a water pump controller,
an industrial PLC — can be deployed in the field with zero open ports, zero
cloud dependency, and zero attack surface, communicating through 186 bytes
that no observer can distinguish from noise. When the protocol is small
enough to run on a €3 microcontroller, simple enough to audit in an
afternoon, and secure enough that compromising one device gives an attacker
nothing beyond that single device.

That is the goal. Everything below serves it.

---

## Primary Use Cases (aside from typical port knocking in sysadmin)

### 1. Offline farm networks

Rural farms — often hundreds of hectares — lack cellular coverage and
reliable internet. Existing solutions are either unencrypted LoRa (readable
by anyone with a €20 SDR), or cloud-dependent IoT platforms with monthly
fees and vendor lock-in.

Sknock enables a self-contained farm network:

- **Soil moisture sensors** scattered across fields send encrypted readings
  via LoRa to a central gateway. No internet required. No cloud. No
  subscription.
- **Irrigation actuators** receive encrypted commands from the gateway to
  open or close valves. Only the authorized gateway can issue commands.
- **Weather stations** report temperature, humidity, pressure, wind — all
  authenticated and tamper-proof.

The entire network is invisible. A neighboring farm, a curious passerby, or
a malicious actor with a radio sees nothing but noise on the frequency.
Nobody can inject false sensor readings to trigger unnecessary irrigation.
Nobody can open a valve remotely.

**Hardware cost**: ~€10-15 per node (ESP32 + LoRa + sensor), ~€40 for a
gateway (Raspberry Pi + LoRa). A 25-node farm network costs under €500 with
no recurring fees.

**Why sknock over alternatives**: existing farm IoT (The Things Network,
Sigfox, commercial platforms) either transmit in cleartext, require internet
backhaul, charge per message, or all three. Sknock provides encryption,
authentication, and offline operation in a single protocol that fits in one
LoRa packet.

### 2. IoT device security

The current state of IoT security is broken. Millions of devices ship with
open ports, default credentials, unencrypted communication, and firmware
that never gets updated. Botnets like Mirai exploited this at scale in 2016
and the situation has not fundamentally improved.

Sknock addresses the root problem: **attack surface**.

A device running sknock has no open ports. It does not respond to any
network traffic. To a port scanner, it does not exist. The only way to
interact with it is to possess the correct cryptographic credentials and
produce a valid 186-byte packet.

This applies regardless of transport:
- **WiFi devices** in a home or office: invisible to Shodan, resistant to
  network-level attacks even if the WiFi password is compromised
- **Ethernet devices** in industrial networks: zero attack surface on the
  network segment, even with flat network topology
- **LoRa/radio devices**: communication content is encrypted, device
  identity is unlinkable between packets

The security model is defense-in-depth:
1. No open ports (nothing to attack)
2. ECIES encryption (nothing to read)
3. Per-packet forward secrecy (nothing to decrypt retroactively)
4. HMAC authentication (nothing to forge)
5. Counter-based replay protection (nothing to reuse)
6. Per-device minimum privilege (nothing to escalate)
7. Counter collision detection (compromise is detectable)

**Why sknock over alternatives**: existing IoT security solutions (TLS
client certificates, MQTT+TLS, cloud IAM) all require the device to have an
open port and a functioning TCP/IP stack. They protect the channel but leave
the service exposed. Sknock eliminates the service entirely — there is
nothing to connect to.

### 3. Critical infrastructure protection

Water treatment plants, electrical substations, gas pipeline monitors, dam
sensors — these systems increasingly use networked devices for monitoring
and control. Many run legacy SCADA protocols with no encryption, no
authentication, and known vulnerabilities.

Regulatory pressure is mounting:
- **EU NIS2 Directive**: requires essential service operators to implement
  risk-appropriate security measures for network and information systems
- **US CISA advisories**: repeatedly highlight vulnerabilities in
  water/energy SCADA systems
- **IEC 62443**: industrial cybersecurity standard requiring defense-in-depth

Sknock fits as a hardening layer for critical infrastructure:

- **Remote monitoring stations** (water level, pressure, flow) communicate
  readings via sknock. No open ports, no exposed SCADA protocols.
- **Maintenance access**: field technicians knock to temporarily open SSH or
  a management interface. The port closes automatically after a
  configurable timeout. No VPN infrastructure required.
- **Inter-site communication**: between substations or pump stations on
  private networks, sknock provides authenticated, encrypted signaling
  without the complexity of PKI or certificate management.

The zero-response design is particularly valuable here: a determined attacker
scanning the network of a water plant finds no services, no banners, no
version strings, no attack surface. The monitoring system is invisible.

**Why sknock over alternatives**: traditional SCADA security relies on
network segmentation (VLANs, firewalls) and VPNs. These work but are
complex to deploy in remote locations with limited connectivity and no IT
staff on-site. Sknock provides endpoint-level protection that works
independently of network architecture — a single static binary on each
device, configured once.

---

## What this is NOT

- **Not a replacement for TLS/HTTPS** in environments with stable internet
  and established infrastructure. If you have a cloud provider and a DevOps
  team, use their tools.
- **Not a C2 framework**. The protocol is unidirectional by design and
  optimized for authentication, not for covert operations.
- **Not a mesh networking protocol**. Sknock defines how two endpoints
  authenticate and communicate securely. Mesh routing, message relay, and
  network topology are separate concerns.
- **Not military-grade communications**. While the cryptography is strong
  (X25519, AES-256-GCM, HMAC-SHA256), the project has not undergone formal
  security audits or certification processes required for defense use.

---

## Design Principles

1. **One packet is the unit of communication.** Everything fits in a single
   self-contained packet — as small as 93 bytes for a knock-only signal, or
   larger for commands and telemetry. The packet size adapts to the transport
   (93 bytes for long-range LoRa, 186 bytes for UDP, larger for data-rich
   use cases). No fragmentation, no reassembly, no multi-packet protocols.

2. **The device does not exist until it chooses to.** No open ports, no
   responses, no acknowledgements. Existence is a choice, not a default.

3. **The protocol is the product.** The value is in the 186-byte packet
   format and its cryptographic properties, not in any specific transport
   or implementation language.

4. **Simple enough to audit, small enough to embed.** The core protocol
   should be implementable in ~300 lines of C, auditable by a single
   security engineer in a day, and runnable on a €3 microcontroller.

5. **Compromise of one device yields nothing beyond that device.** Each
   device has its own credentials, its own permissions, its own restricted
   scope. There are no master keys, no shared secrets across devices, no
   lateral movement paths.
