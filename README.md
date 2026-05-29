# Frostcast

**CS1680 Final Project — Eric Xu and James Yan**

An extension of the Snowcast audio streaming platform, rebuilt with gRPC and adaptive bitrate streaming.

---

## Overview

This project extends Snowcast to use gRPC instead of raw TCP for client-server communication, and adds adaptive bitrate streaming where clients automatically switch between quality tiers based on local network conditions.

---

## Features

### gRPC Protocol

All message types (Hello, SetStation, Welcome, InvalidCommand, etc.) are defined in a `.proto` file and compiled via the protobuf compiler into `protocol_grpc.pb.go` (networking) and `protocol.pb.go` (message structs).

The `SetStation` RPC uses a server-side stream so the server can continuously push `AnnounceMessage` and `InvalidCommandMessage` events to the client over an open connection, wrapped in a `ServerEvent` message using gRPC's `oneof` operator.

**RPCs:**
```protobuf
service SnowcastControl {
    rpc Handshake(HelloMessage) returns (WelcomeMessage);
    rpc SetStation(SetStationMessage) returns (stream ServerEvent);
    rpc Disconnect(DisconnectRequest) returns (DisconnectResponse);
}
```

### ListStations RPC

A new `ListStations` RPC allows clients to query available stations and their supported bitrate levels directly from the server. Accessible via the client CLI by typing `l`:

```
Station 0: music/song1.mp3 [bitrates: low, medium, high]
Station 1: music/song2.mp3 [bitrates: low, medium, high]
```

### Adaptive Bitrate Streaming

The server supports three quality tiers, sending different chunk sizes per tick based on each client's current bitrate:

| Bitrate Level | Chunk Size | Approx. Rate |
|---|---|---|
| low | 375 bytes | ~32 KB/s |
| medium | 750 bytes | ~64 KB/s |
| high | 1500 bytes | ~128 KB/s |

Tick interval is fixed at ~91.5ms. Multiple clients on the same station can independently receive different quality levels.

**Manual switching:** Type `0 low` to connect to station 0 at low quality. Typing just `0` defaults to high quality.

**Automatic switching:** A background goroutine (`autoSwitch`) opens a UDP socket on the listener port and monitors throughput every 3 seconds, switching tiers according to:

| Observed Throughput | Current Bitrate | Action |
|---|---|---|
| < 3.0 KB/s | medium or high | Switch to low |
| 3.0 – 10.0 KB/s | high | Switch to medium |
| >= 3.5 KB/s | low | Switch to medium |
| >= 7.5 KB/s | medium | Switch to high |

A `flushMetrics` flag discards one measurement window after any switch to prevent stale data from triggering an immediate follow-up switch.

**Client-side metric logging:** The controller reports throughput and current bitrate every 3s to stderr. The listener reports throughput, average chunk arrival interval, and buffer health (flagged as "degraded" if arrival interval deviates significantly from the expected ~91.5ms tick).

---

## Setup

```bash
./setup.sh
```

This downloads all additional dependencies not present in the original project.

---

## Running Tests

```bash
./run-tests.sh
```

New unit tests cover each RPC function and the handshake. All tests pass.

---

## Results

Under simulated network restriction (`sudo dnctl pipe 1 config bw 20Kbit/s`), the client detected sub-3KB/s throughput within one measurement window and automatically switched to low bitrate. After removing the restriction, throughput recovered and the client stepped back up to medium then high over two windows.

The 3-second measurement window was chosen to prevent rapid tier oscillation under fluctuating network traffic — a known failure mode observed during development.

---

## Stack

- Go
- gRPC + Protocol Buffers
- UDP (listener)
- Linux `tc` / `dnctl` (traffic shaping for testing)