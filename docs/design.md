# gige-vision3 — Initial Design

Status: proposal for review. Design only; no library code in this PR.
Source: *GigE Vision Specification v3.0.00* (A3, 2026-04-24), referred to as "the spec".
Citation convention: `§` is a spec section/chapter, `p.N` is the **PDF page** (printed page number = PDF page − 1). Requirement IDs (`R-xxx`, `CR-xxx`, `O-xxx`) are quoted from the spec so they can be traced.

## 1. Purpose and framing

`gige-vision3` is a public Rust library for talking to GigE Vision devices (cameras and other peripherals) from a host. It is **application-side**: it plays the "application" role of §1.7 (p.28–30) — discover devices, take control, configure, receive streams, handle events. Implementing the *device* side (a transmitter) is out of scope for the library proper, except for a **simulated device** used in tests (§9).

Legal note: the spec says commercial products must license the standard and pass compliance testing before using the GigE Vision name/logo (p.2). The spec is copyrighted and may not be reproduced (p.2), so this repo must paraphrase, not paste, spec text and tables. See open questions.

## 2. Scope: what v3.0 contains and what we cover

| Part | Spec | Topic | v1? |
|---|---|---|---|
| 1 | Ch.2–6, p.32–52 | Link config (single, multi-link, LAG), IP config (persistent/DHCP/LLA), enumeration, attach/removal | Partial: see below |
| 2 | Ch.7–21, p.57–162 | GVCP: control channel, message channel, commands, status codes, events, ICMP | **Yes** (core) |
| 3 | Ch.22–26, p.164–205 | GVSP: UDP streaming, data blocks, payload types, pixel formats | **Yes** (core) |
| 4 | Ch.27–35, p.206–245 | GVRSP: new in 3.0, streaming over RoCEv2 (RDMA); GenDC only | **No** (v2, optional feature) |
| 5 | Ch.36–37, p.246–364 | Bootstrap registers; standard camera features list | **Yes** (registers); SFNC lookup is data only |
| – | Ch.38–39, p.365–368 | GenDC usage; testing (required test features) | Ch.39 yes (test features inform our simulator) |

Feature-level scope:

- **Device discovery** (§5, p.48): GVCP `DISCOVERY` broadcast and unicast, plus `FORCEIP` (§16.2, p.123). Both are mandatory device commands (R-139cd, R-147cd). **v1.**
- **Zeroconf (mDNS / DNS-SD)** (§5.2, p.49–): optional on devices; lets one host expose several GVCP services on non-3956 ports. **v1.1** behind a feature flag (needs a multicast DNS dependency). Without it we cannot reach multi-service devices.
- **GVCP control channel** (Ch.10, p.69–78): privilege (exclusive / control / control-with-switchover / monitor), CCP register open/close, heartbeat, pending-ack, req_id retry. **v1.** Switchover (§14.4, p.109) is **v1.1**.
- **Register and memory access:** `READREG`, `WRITEREG`, `READMEM`, `WRITEMEM` (§16.3–16.6). **v1.**
- **GenICam / XML feature access:** retrieve the device description file via URL registers 0x0200/0x0400 or the Manifest Table (§14.1, p.94–98; R-118ca requires both plain XML and ZIP, DEFLATE/STORE only). **v1 retrieves, unzips and caches the file.** Evaluating the XML (GenApi) is a hard problem; see §6.
- **GVSP streaming** (Ch.22–25): leader/payload/trailer blocks, 64-bit block IDs, packet-size negotiation, `PACKETRESEND`, flow-control knobs. Payload types in v1: **Image (0x0001), Chunk data (0x0004), GenDC (0x000B)** — the three the 3.0 spec itself redefines (Table 25-1, p.186). Others (JPEG, JPEG 2000, H.264, Multi-part) are defined only in the 2.2 spec, which we do not have: **v1 delivers their raw blocks undecoded**.
- **Message channel / events** (Ch.12 p.88–, Ch.17 p.147, Ch.20 p.160): `EVENT_CMD`, `EVENTDATA_CMD`, standard event IDs. **v1.**
- **Action commands** (§14.3, p.103–109; §16.9): trigger multiple devices; scheduled variant needs IEEE 1588 time. Unscheduled action: **v1.1**. Scheduled: **v2**.
- **IEEE 1588 / PTP** (§14.2, p.98–): configure device clock via registers only; we do not run a PTP stack. **v2.**
- **Multi-link and LAG** (Ch.3, p.34–; Ch.13 p.93): v1 supports a device with several interfaces only by treating each IP as an independent device (matches 1.x behavior, p.50). LAG is transparent to us except for events (§3.3.6). Real multi-link stream balancing: **v2**.
- **Multicast streams** (§11.9, p.86): v1.1.
- **Firewall/NAT stream hole punching** (§11.11, p.86–87; SCSPx): v1.1.
- **GVRSP / RoCEv2** (Parts 4): v2 or later, behind a feature flag. Requires an RDMA verbs stack (Linux rdma-core / libibverbs); it has no Windows/macOS story. Also needs connection management over RDMA-CM (§31, p.~230).
- **Out of scope forever (this crate):** device-side firmware, GigE Vision compliance certification, GUI.

## 3. Crate and module layout

A Cargo workspace; the root `gige-vision3` crate re-exports the useful pieces.

```
gige-vision3/                     (workspace root; facade crate re-exporting)
  crates/
    gvcore/        no_std + alloc. Wire formats and pure logic. No I/O.
      gvcp/        header, command/ack encode+decode, status codes, flags
      gvsp/        packet header, leader/trailer per payload type, test-packet LFSR
      bootstrap/   register address map, typed accessors, capability bitfields
      reassembly/  block assembler + missing-packet tracker (sans-IO)
      error.rs
    gvnet/         std. Sockets, timers, threads/tasks.
      discovery/   broadcast/unicast discovery, FORCEIP, (feature "mdns")
      control/     ControlChannel: req_id, retry, pending-ack, heartbeat
      stream/      StreamReceiver: socket(s), resend loop, packet-size probe
      message/     MessageChannel: events
      xml/         description-file fetch + unzip + cache
    gvxml/         (v1.1) GenApi evaluation; or adapter to an external crate
    gvsim/         simulated device for tests (dev-dependency only, not published)
    gvcli/         example tool: discover / info / grab (not part of public API)
```

Rationale: wire code is the part most worth fuzzing and reusing, so it stays pure and `no_std`; sockets and clocks are isolated in `gvnet` so the state machines can be tested deterministically ("sans-IO").

### Public API sketch (illustrative, not final)

```rust
// Discovery
let devices: Vec<DeviceInfo> = discover(DiscoverOptions::default()).await?;
let info = &devices[0]; // mac, ip, manufacturer, model, serial, user_name, spec_version, ...

// Control
let dev = Device::open(info, Access::Control /* or Exclusive, Monitor */).await?;
let v: u32 = dev.read_reg(Reg::GvcpCapability).await?;
let bytes = dev.read_mem(addr, len).await?;
let xml = dev.description().await?;            // fetched, unzipped, cached

// Streaming
let mut rx = dev.open_stream(0, StreamConfig { packet_size: Auto, resend: ResendPolicy::default(), .. }).await?;
while let Some(block) = rx.next_block().await {
    match block { Ok(Block { id, payload: Payload::Image(img), .. }) => {...}
                  Err(BlockError::Incomplete { id, missing, .. }) => {...} }
}

// Events
let mut ev = dev.open_events().await?;
while let Some(e) = ev.next().await { /* Event { id, timestamp, data } */ }
```

Principles: typed registers (no magic addresses in user code); streaming yields **blocks with a status**, never silently-corrupt images; every wire struct has `encode`/`decode` that is total (never panics on hostile bytes).

## 4. Sync vs async, networking and reassembly

**Decision: the protocol core is sans-IO; the I/O layer ships async-first on Tokio, with a thin blocking facade behind a `blocking` feature.** Reasons: GVCP is strict request/response with timers (§8.1.3, p.63–64), GVSP is high-rate UDP that benefits from a dedicated receive thread, and users of vision libraries often run in plain threaded apps. Alternative considered: threads-only std. See open questions.

### GVCP (control and message channels)
- UDP, big-endian (R-137c, p.115), header byte 0 = `0x42`, 16-bit command/length/req_id (§15, p.113–117). Default max packet 576 bytes to avoid IP fragmentation (§8.1.2, p.63); the **Control Channel Max Packet Size** register (CCMPS) may allow more.
- **One outstanding command at a time** when an ack is requested (§8.1.4, p.64–65). The client serialises commands per device.
- `req_id` increments, wraps 65535 → 1, never 0 (p.114). A retry reuses the same `req_id`; the device must not re-execute but must re-ack (R-051cd, R-052cd, p.63). Acks whose `ack_id` mismatches are silently discarded (Sorcerer's Apprentice, p.64).
- Timeout and retry counts come from the bootstrap registers; `PENDING_ACK` (0x0089) extends the wait (§10.7, p.76).
- **Heartbeat:** primary app must keep the device's heartbeat alive; recommended three pings per timeout period (default 3 s) (§8.1.5 p.65, R-076cd p.75). A background task reads CCP periodically. Note `PACKETRESEND`, `DISCOVERY`, `ACTION`, `FORCEIP` do **not** reset the heartbeat (R-075cd).
- **Open/close:** write CCP to take privilege; **write 0 on close** (R-071ca p.74). On drop, `Device` closes best-effort; on hard crash the device times out.

### GVSP (streaming, UDP)
- Packet header: status(16) | flags(16) | EI+format | block_id64 | packet_id32 (§24.2, p.173–174). Block = leader (id 0) → payload packets → trailer. No length field; UDP length is used (p.173). Leader/trailer ≤ 576 bytes unless `large_leader_trailer_enable`.
- **Packet-size negotiation** via the test packet (§11.7, p.84–85): write SCPSx with fire-test and do-not-fragment bits, binary search. Must happen after the channel is opened and before regular streaming (R-567s).
- **Receive path:** one receive thread/task per stream channel, large `SO_RCVBUF`, `recvmmsg` on Linux (optimisation, later). Place payload by `packet_id32 × payload_size` directly into a pooled buffer (p.176–177).
- **Reassembly** (sans-IO state machine in `gvcore::reassembly`): track highest packet id, a bitmap of received packets, block-level deadline. Out-of-order is expected (p.165). Detect gaps → issue `PACKETRESEND_CMD` (command 0x0040) with `first_packet_id`/`last_packet_id` (0xFFFFFFFF = to trailer) (§16.7, p.136–138). Resend request has no ack: success is proven by the resent packets arriving, flagged `GEV_FLAG_PACKET_RESEND` (CR-204st). A resend that cannot be served arrives as header-only packet with a status such as `PACKET_UNAVAILABLE` / `TEMPORARILY_UNAVAILABLE` (carries a hold-off hint) (CR-205st, Table 16-2, p.139) → we back off or drop the block.
- Resend only if the device advertises the capability bit (bit 29 of GVCP Capability at 0x0934, R-197ca).
- Policy knobs: max resend attempts, resend delay, block timeout, buffers in flight, drop-oldest vs backpressure. Block statuses: Complete, Recovered, Incomplete (missing ranges), DeviceError(status), PreviousBlockDropped (flag 14).
- Flow-control knobs we expose but do not auto-tune in v1: inter-packet delay (SCPDx), 802.3 PAUSE config (§23.1.4, p.167).
- Test packets (block_id 0) carry LFSR data; we implement the generator for self-test (§24.5, p.183).

### Message channel
- Device-initiated EVENT/EVENTDATA over UDP to a host port written into MCP (§12.2). The host must ack using the source port of the incoming packet (CR-100ca). Retry and timeout registers exist (MCTT, MCRC). Event IDs per Table 20-1, p.160 (e.g. 0x0007 primary-app switch, 0x0008 link speed change, 0x0009 action late, 0x8001–0x8FFF async errors, 0x9000+ device specific). Device-specific events are interpreted via the XML (§17.3).

## 5. Error model

- `gvcore`: `DecodeError` (truncated, bad key byte, bad length, reserved bits ignored per spec rules). Pure, `Copy`-friendly, `#[non_exhaustive]`.
- `gvnet`: one `Error` enum: `Io`, `Timeout { op, retries }`, `Device(Status)` (a typed `Status` for every code in Ch.19, p.156: SUCCESS 0x0000, NOT_IMPLEMENTED 0x8001, INVALID_PARAMETER, INVALID_ADDRESS, WRITE_PROTECT 0x8004, BAD_ALIGNMENT, ACCESS_DENIED, BUSY, …, with an `Unknown(u16)` for forward compatibility and device-specific ranges), `AccessDenied`, `NotSupported(Capability)`, `Protocol(..)` (unexpected ack, wrong ack_id).
- `ACCESS_DENIED` acks must have length 0 and consumers ignore any trailing data (R-138cd, p.119).
- Streaming never returns an error for lost data on the iterator path; it returns `Err(BlockError)` per block so the stream continues.
- Use `thiserror`; implement `std::error::Error`; no panics on network input; fuzz decoders.

## 6. GenICam / XML

The spec treats GenICam as the way to know a device's real features; the wire protocol only gives bootstrap registers. v1 deliverables: fetch (device memory `local:` URL via READMEM, `http(s):`, `file:`; Manifest Table preferred when present, R-121ca/R-122ca p.98), unzip, cache by unique filename (p.96), expose bytes + parsed manifest. **Unknown:** the spec does not define the GenApi evaluation rules (those are the separate GenICam standard); **we do not have that document and no mature pure-Rust GenApi crate is known to us** — treat as unknown pending research. Options: (a) write our own minimal GenApi evaluator (large), (b) FFI to the GenICam reference implementation (C++, licensing differs), (c) leave feature access at the register layer plus a SFNC-name→register helper for a few standard cameras. **Recommend (c) for v1, (a) as a separate crate later.** Chunk-data decoding also depends on GenICam (§24.4.2, p.182): v1 returns chunk id+bytes.

## 7. no_std and platform considerations

- `gvcore`: `#![no_std]` + `alloc` (optionally no `alloc` for header codecs). Enables embedded device-side reuse and WASM test builds.
- `gvnet`: `std`. Linux first-class; macOS and Windows supported for GVCP/GVSP (plain UDP). Raw-socket tricks are avoided. Platform specifics: Linux `SO_RCVBUF` caps via `net.core.rmem_max` (document it); Windows needs firewall rules; broadcast on multihomed hosts is ambiguous (spec admits it defines none, §5.1.1, p.49) → we enumerate interfaces and send **subnet-directed broadcasts per interface**, with explicit `bind` per interface. IPv4 only: the spec's transport is "UDP IPv4" (§22.1, p.164).
- Jumbo frames are needed for good throughput; we must discover MTU via the test packet, not assume it.
- GVRSP: Linux-only, behind feature `gvrsp`.
- MSRV: latest stable; edition 2024 (already set in `Cargo.toml`).
- Licensing: MIT (existing). Check dependency licences.

## 8. Dependencies (proposed, minimal)

`tokio` (net, time, sync), `bytes`, `thiserror`, `tracing`, `socket2` (buffer sizes, per-interface bind), `zip`/`flate2` (XML), `bitflags`, `if-addrs` or similar (interface enumeration), `mdns-sd` (feature). Dev: `proptest`, `cargo-fuzz`, `criterion`.

## 9. Testing strategy

1. **Golden vectors:** the spec gives an example WRITEREG_CMD byte layout (§15.3, p.116–117: `42 01 00 82 00 08 03 3A 00 00 0D 04 C0 00 21 9C`). Unit-test encode/decode against it, plus hand-built vectors per command.
2. **Property tests / fuzzing** of all decoders (no panics, round-trips). Hostile-input corpus.
3. **Reassembly tests** in the sans-IO core: shuffled, duplicated, dropped, late packets; resend flows; block-id gaps; zero-payload blocks (valid per p.175).
4. **Simulated device (`gvsim`)**: a UDP responder on loopback implementing: DISCOVERY/FORCEIP, bootstrap register file, CCP/privileges/heartbeat timeout, READ/WRITE REG/MEM with the 576-byte limit, PENDING_ACK, duplicate `req_id` handling, XML in memory (plain and ZIP), one stream channel (test packet, Image and Chunk blocks, configurable packet loss/reordering/duplication, `PACKETRESEND` support and its error statuses), message channel with events, action commands. It includes **fault injection** (drop/delay/duplicate GVCP acks) so the Sorcerer's Apprentice and timeout paths are exercised. It also mirrors the spec's required test features (Ch.39 p.367–368: `TestPendingAck`, `TestEventGenerate`, `TestPayloadFormatMode`) so we can later write conformance-style tests.
5. **Integration tests** run against `gvsim` in CI on Linux, macOS, Windows (loopback, no hardware).
6. **Hardware smoke tests:** opt-in (`--ignored`), documented, run manually against real cameras. **Unknown/needed:** access to a camera, see open questions.
7. **Interop reference:** compare captured pcaps from known-good tools (e.g. Wireshark GVCP/GVSP dissectors) to our encoders.
8. Compliance: A3 compliance is self-certified (§39.1, p.367) and relies on a "Validation Framework" we do not have; we do not claim compliance.

## 10. Milestones (build order)

- **M0 — Scaffold:** workspace, CI (fmt, clippy, test, `no_std` check, Linux/macOS/Windows), licence/NOTICE, contribution notes.
- **M1 — `gvcore` GVCP wire:** header, status codes, all commands in Ch.16–17 (encode/decode), golden + fuzz tests.
- **M2 — Bootstrap register map** (Ch.36, p.246–): typed registers/capability bits; address table generated from a checked-in data file.
- **M3 — `gvsim` v0:** discovery + register file + CCP, enough to test M4.
- **M4 — Discovery and control:** `discover`, `FORCEIP`, `Device::open`, req_id/retry/pending/heartbeat, reg/mem access, XML fetch.
- **M5 — `gvcore` GVSP wire + reassembly:** packet formats for Image/Chunk/GenDC, block assembler, resend planner (all sans-IO) with property tests.
- **M6 — `gvnet` streaming:** receive loop, packet-size negotiation, resend, block API; `gvsim` stream with loss.
- **M7 — Events** (message channel) and `gvcli grab`.
- **M8 — Hardening:** perf (`recvmmsg`, buffer pools), docs, examples, `v0.1.0`.
- **Post-v1:** mDNS, action commands, multicast, firewall punching, switchover, blocking facade polish, GenApi crate, multi-link, PTP config, then GVRSP.

## 11. Unknowns and gaps in the spec (as read)

- **Page numbers/IDs in the spec TOC and body differ by one**; we cite PDF pages.
- GenApi evaluation semantics: not in this document (referenced external standard). **Unknown.**
- JPEG / JPEG 2000 / H.264 / Multi-part payload layouts: defined only in 2.2 (p.186 note). **Unknown** from this document.
- The default *transmission timeout and retry values* for the control channel (bootstrap register defaults) — not re-checked in this pass; see Ch.36 before M4. **Unknown (to verify).**
- Exact behaviour when a host has several NICs for broadcast discovery: spec explicitly leaves open (p.49).
- How a receiver learns that a test packet arrived in multi-receiver topologies: left to implementers (p.85).
- Not yet read in depth during this pass (flagged for the next pass): Ch.3 link/LAG details, Ch.4 IP config, Ch.6 attach/removal, §14.4 switchover, Ch.26 pixel formats, GVRSP Parts 4 in full, all of Ch.36 field definitions.

## 12. Open questions for the human

1. **Licence/legal posture:** the spec is copyrighted and commercial use requires licensing (p.2). Is this library intended to be *used in* licensed products, and may we describe protocol structures (field layouts) in code/docs? Do we want a "not certified" disclaimer?
2. **Async runtime:** Tokio-first with a blocking facade (proposed), or runtime-agnostic / threads-only?
3. **GenICam strategy:** own GenApi crate, FFI to the reference implementation, or register-level only for v1? Do you have the GenICam standard documents?
4. **Hardware access:** which camera(s), NIC(s) and OS(es) are available for validation? Do we have the Validation Framework?
5. **GVRSP priority:** is RoCEv2 streaming wanted at all (Linux + RDMA NICs only), or defer indefinitely?
6. **Scope of role:** application-side only (proposed), or also a reusable device-side/transmitter implementation?
7. **Workspace split and publishing:** multi-crate workspace vs single crate with features; crate names on crates.io; MSRV policy.
8. **Older devices:** require GigE Vision 2.x/3.0 devices only, or support 1.x quirks (16-bit block IDs — the spec drops them, p.174)?
