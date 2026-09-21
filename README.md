# NetBeam

NetBeam is the Airdrop for CodeOS. It lets two CodeOS machines discover each
other on the same LAN (via UDP beacons) and exchange files directly using a
lightweight TCP transfer protocol — no central server.

Each release works against a real kernel network stack (IP / UDP / TCP) that
is part of the CodeOS kernel. The sources in this repo mirror the current
in-tree implementation:

```
app/
  netbeam.cpp          Qt 6 GUI app: tabbed UI, device discovery, file picker,
                       send/receive threads (portable against the CodeOS
                       syscall surface: tcp_*, udp_*, fs_*, sched_*)
kernel/
  ip.c                 IP stack (checksum validation, poll-driven ARP
                       resolution, route lookup)
  udp.c / udp.h        UDP stack (checksum validation, discovery beacons,
                       non-blocking poll mode, udp_get_local_port)
  tcp.c / tcp.h        TCP stack (3-way handshake, ordered stream, windowing,
                       net_poll-driven inbound processing)
```

## Kernel stack notes (current in-tree behaviour)

* **Poll-driven ARP.** `ip.c` does not rely on a background netd thread to
  drain the NIC and feed the ARP cache. It consumes inbound ARP replies
  itself (`nic_recv`), with a ~0.3–3s bounded wait for a reply to arrive, so
  cold-cache resolution (e.g. in VirtualBox) completes even when the peer
  answers late.
* **Non-blocking UDP poll.** `udp_recv_timeout` with `timeout_ms == 0` means
  "poll once": it returns immediately (`-1`) when no datagram is pending
  instead of sleeping for the full timeout.
* **`udp_get_local_port`.** Returns the bound local port of a UDP socket,
  falling back to a deterministic ephemeral port (`49152 + fd*137 + 42`)
  when the caller has not bound one.
* **`net_poll()` in TCP paths.** The connect/accept paths call `net_poll()`
  so inbound SYN-ACK / ACK segments are processed regardless of netd
  scheduling.

## Protocol

* Discovery — UDP port `54917`, broadcast beacon:

  ```
  NB1|<machine-name>|54918
  ```

* Transfer — TCP port `54918`, connection established after discovery:

  ```
  NB1|<filename>|<size>\n      ← control line, then raw file bytes
  ```

Verify end-to-end inside CodeOS:

```
A: open NetBeam → click peer card → choose file → Devices → card
B: NetBeam logs:  NBRECV ok file=<name> size=<n> saved=/NetBeam/Incoming/<name>
```

## License

GPL-3.0 (see LICENSE), matching the CodeOS kernel.