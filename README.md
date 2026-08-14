# MIC-TCP

> Stop-and-wait TCP implementation in C with tunable loss tolerance, for video over lossy links.

## About

MIC-TCP is a custom transport protocol implemented in C on top of UDP, built for the *BE Réseau* lab course (3 MIC) at INSA Toulouse. It provides a TCP-like socket API (`mic_tcp_socket`, `mic_tcp_bind`, `mic_tcp_connect`, `mic_tcp_accept`, `mic_tcp_send`, `mic_tcp_recv`, `mic_tcp_close`) with a three-way handshake and stop-and-wait reliability, and is tested by streaming video through VLC over a link with simulated packet loss. All four assignment versions were implemented, up to v4.1.

The key feature is **partial reliability with negotiated loss tolerance**:

- **Stop-and-wait transfer** — each data PDU (alternating sequence numbers 0/1) is sent, then the sender waits up to 100 ms for an ACK before deciding whether to retransmit.
- **Partial reliability (v3)** — a sliding window (`histPDU`, last 10 packets) tracks the observed loss rate; a lost packet is only retransmitted if that rate exceeds the negotiated maximum acceptable loss `max_perte`. This trades a few lost frames for smoother video, which matters more than perfect delivery.
- **Loss-rate negotiation (v4)** — during connection establishment, the server piggybacks its `max_perte` proposal on the SYN-ACK and the client returns the agreed value in the final ACK. If the client's proposal is lower than the server's, the client's value wins; otherwise the two are averaged. Since the client is the one requesting the service, it gets the final word, while the server still gets a say. (The two proposals are randomized with `rand()` to illustrate the negotiation without modifying `client.c`/`server.c`.)
- **Sequence/ACK tracking** — earlier versions used `next_seq_num`/`seq_attendu` variables; v4 consolidates both sides into a two-slot `seqTracker[]` (index 0 for the source, 1 for the sink).

Compared against plain TCP on a high-loss link (20–50% simulated loss), MIC-TCP delivers noticeably smoother video playback, since it does not stall retransmitting frames the viewer would never miss.

## Tech stack

- C (`gcc -std=gnu99`), POSIX threads
- UDP sockets as the underlying transport (`src/api/mictcp_core.c`, with configurable simulated loss via `set_loss_rate`)
- GNU Make, Bash launcher scripts
- VLC (`cvlc`) for the video streaming demo
- GitHub Actions CI (build on push/PR)

## Build & run

```sh
make clean; make
```

This produces `build/client`, `build/server` and `build/gateway`.

Text transfer test (start the sink first, then the source):

```sh
./tsock_texte [-p|-s destination] port
```

- `-p` runs the sink (server), `-s destination` runs the source (client).

Video streaming test (requires VLC; run the sink and the source in two terminals):

```sh
./tsock_video [-p|-s] [-t (tcp|mictcp)]
```

- `-t` selects the transport: plain `tcp` or `mictcp`, so the two can be compared under loss.

## Project structure

```
include/            MIC-TCP API and PDU/socket type definitions
  api/              Core API header (IP_send/IP_recv, app buffer, set_loss_rate)
src/
  mictcp.c          The protocol itself: handshake, negotiation, stop-and-wait, partial reliability
  api/mictcp_core.c Provided UDP-based core with simulated packet loss
  apps/             Test applications: client, server, video gateway
video/              Sample video files for the streaming demo
tsock_texte         Launcher script for the text client/server
tsock_video         Launcher script for the VLC video comparison demo
Makefile            Builds client, server and gateway into build/
```

## Notes

- Getting the server and client to use two genuinely different `max_perte` suggestions without touching `client.c`/`server.c` was awkward, hence the `rand()`-based proposals; race conditions between the protocol threads make it hard to pause for user input. V4 appears more stable than v4.1, but both work well.
- In v4 the server does not always finish initializing before the client sends its first PDU; the client immediately retransmits the lost packet so this is harmless, but it is likely another thread race.
- A possible improvement: derive the acceptable loss rate from the video's frame rate — the eye only needs 24–30 fps for smooth motion, which would make a good loss threshold easier to predict.
