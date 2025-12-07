# CAN logging start modes

The `can log start` command family creates loggers that either export CAN
activity or accept CAN frames from remote peers. Each transport subcommand
uses the same filter syntax (`<bus> | <id>[-<id>] | <bus>:<id>[-<id>]`) and
shares the available record formats (`crtd`, `cs11`, `gvret-a`, `gvret-b`,
`lawicel`, `panda`, `pcap`, `raw`).

### Record formats

- **crtd**: human-readable rows beginning with an epoch-based timestamp, the
  bus number and direction (`R` or `T`), followed by an 11-/29-bit flag, the
  message ID, and each data byte in hex. Error/status entries add counters and
  queue depth, while info records carry text payloads.【F:vehicle/OVMS.V3/components/can/src/canformat_crtd.cpp†L57-L137】
- **cs11**: CANswitch binary frames where the first byte encodes payload length
  (data bytes plus three control bytes). The second byte contains the bus
  index, the next two bytes hold the 11-bit standard ID (LSB first), and the
  remaining bytes carry the CAN payload. Incoming `len==0` or `len==255`
  values signal sync and bitrate commands instead of frames.【F:vehicle/OVMS.V3/components/can/src/canformat_canswitch.cpp†L63-L140】
- **gvret-a**: ascii GVRET lines with microsecond timestamps, hexadecimal ID,
  a standard/extended marker (`S`/`X`), bus number, DLC, and space-separated
  byte values, each line terminated by `\n`.【F:vehicle/OVMS.V3/components/can/src/canformat_gvret.cpp†L90-L113】
- **gvret-b**: binary GVRET packets as implemented by SavvyCAN, decoded and
  served by the same handler as the ASCII variant; inbound parsing populates
  CAN frames while outbound support is intentionally omitted in the OVMS
  logger.【F:vehicle/OVMS.V3/components/can/src/canformat_gvret.cpp†L114-L177】
- **lawicel**: LAWICEL/SLCAN ASCII lines starting with `t` or `T`, followed by
  a zero-padded 3/8-digit ID, DLC, concatenated data bytes, and a 4-digit
  millisecond timestamp before the trailing newline.【F:vehicle/OVMS.V3/components/can/src/canformat_lawicel.cpp†L63-L118】
- **panda**: comma.ai Panda binary packets (12 bytes) with a 21-bit identifier
  in `w1`, a DLC/bus nibble in `w2`, and up to 8 data bytes. The logger only
  emits Panda frames; inbound packets are ignored.【F:vehicle/OVMS.V3/components/can/src/canformat_panda.cpp†L70-L88】
- **pcap**: classic PCAP records using link type `0xe3` (Linux "can" socket).
  Each captured frame packs timestamp seconds/microseconds, a 32-bit ID/flag
  field with EXT/RTR bits, the DLC, and CAN payload bytes. A global PCAP
  header prefixes the stream.【F:vehicle/OVMS.V3/components/can/src/canformat_pcap.cpp†L60-L101】
- **raw**: raw `CAN_log_message_t` structs written verbatim with the origin bus
  pointer replaced by the bus index. Decoding simply rehydrates the struct and
  restores the origin pointer via `MyCan.GetBus`.【F:vehicle/OVMS.V3/components/can/src/canformat_raw.cpp†L56-L81】

## Serve modes: discard vs simulate vs transmit

Format handling defines what to do with inbound frames:

- **discard**: incoming data is parsed but dropped immediately.
- **simulate**: decoded frames are injected locally via `MyCan.IncomingFrame`
  as if observed on the bus, letting diagnostics run without driving the
  transceiver.
- **transmit**: decoded frames are sent out over the originating CAN bus with
  `origin->Write`, enabling remote control or bridging behaviour.

These behaviours are enforced by the common `canformat::Serve` logic after the
format handler reconstructs `CAN_log_message_t` records.【F:vehicle/OVMS.V3/components/can/src/canformat.cpp†L108-L189】

## monitor

`can log start monitor` streams log output to the module console. The command
accepts filters but no network parameters. Messages are emitted at varying log
levels (frames at verbose, statistics and events at debug, errors at error).【F:vehicle/OVMS.V3/components/can/src/canlog_monitor.cpp†L33-L118】

## tcpclient

`can log start tcpclient <mode> <format> <host:port> [filters]` connects to a
remote TCP server. The client stops immediately if the connection fails and
optionally applies ID or bus filters before forwarding frames. All three serve
modes are available so inbound data from the peer can be dropped, simulated, or
transmitted on the vehicle bus.【F:vehicle/OVMS.V3/components/can/src/canlog_tcpclient.cpp†L40-L113】

## tcpserver

`can log start tcpserver <mode> <format> <host[:port]> [filters]` listens for
incoming TCP connections (defaulting to port `3000` if none is provided) and
serves log data to each client. With simulate/transmit modes, frames received
from clients are fed back into the CAN stack according to the serve policy.
The logger is re-opened automatically when networking becomes available.【F:vehicle/OVMS.V3/components/can/src/canlog_tcpserver.cpp†L41-L188】

## udpclient

`can log start udpclient <format> <host:port> [filters]` sends logs to a UDP
endpoint. UDP has no persistent connection, but the logger still honours
filters and format selection. If a simulate or transmit mode is chosen, any
responses parsed from the peer will be injected or transmitted like TCP, using
the same serve-mode semantics.【F:vehicle/OVMS.V3/components/can/src/canlog_udpclient.cpp†L41-L109】

## udpserver

`can log start udpserver <mode> <format> <port> [filters]` binds a UDP socket
(and prefixes `udp://` automatically when needed) to broadcast logs. Each
client is tracked with a timeout, and incoming frames from clients are handled
per the serve mode just like TCP. This command is useful for stateless
streaming or simulation over LAN.【F:vehicle/OVMS.V3/components/can/src/canlog_udpserver.cpp†L40-L200】

## vfs

`can log start vfs <format> <path> [filters]` writes records to the virtual
filesystem. The logger validates the path, opens a file connection, and keeps
track of file size while applying any filters. Stored logs can be replayed
later with `can play`. This mode never processes inbound frames, so serve-mode
options are not used here.【F:vehicle/OVMS.V3/components/can/src/canlog_vfs.cpp†L38-L143】
