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
  ip.c / udp.c         IP + UDP stack (checksum validation, discovery beacons)
  tcp.c / tcp.h        TCP stack (3-way handshake, ordered stream, windowing)
```

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