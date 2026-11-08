# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **4903**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

I created the loop by modifying handle 2, the `dst_id=3` entry in
`MyIngress.myTunnel_exact` on s2, from port 3 to port 1:

     sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"
     h1 python3 send.py 10.3.0.3 "loop" --tunnel 3

The counts in `evidence/loop-before.txt` increased from 511 packets on each
switch to 1,594 on s1 and 1,595 on s2 during the observation period. The loop
did not stop. `ipv4_forward` decrements the IPv4 TTL, but tunnelled packets use
`myTunnel_forward`, which does not touch the IPv4 header. Therefore the IPv4
TTL stays 64 and cannot stop this tunnel loop. If an IPv4 packet did use
`ipv4_forward`, it would eventually reach zero and be dropped by normal host
or IPv4 processing, but that is not the path used here.

## 2. Design: the hop limit

I added an 8-bit `hop_limit` field after `dst_id` in `myTunnel_t`. Eight bits
allows values from 0 to 255 while keeping the field small; placing it after the
existing routing fields preserves the meaning of `proto_id` and `dst_id`.

`send.py` starts tunnelled packets with a hop limit of 2 by default and exposes
the value through `--hops`. Two is sufficient for the normal two-switch path,
and it makes the loop test easy to observe. `myTunnel_forward` checks the
field before forwarding. A zero value is dropped; otherwise the action sends
the packet and decrements the field. The parser extracts the enlarged header,
the deparser emits it, and both host scripts use the same field order and size.

## 3. Implementation

In `tunnel.p4`, I added the 8-bit field and changed `myTunnel_forward` so that
it drops packets whose limit is zero and decrements packets it forwards. The
existing parser, ingress table, and deparser already handle the tunnel header
from the lab steps, so no new table key was needed.

In `send.py`, `MyTunnel` now includes `hop_limit`, and `--hops` controls its
starting value with a range check from 0 through 255. In `receive.py`, the
same field is parsed and printed so the evidence shows the value at delivery.
The P4 compiler and Python syntax checks passed before the network tests.

## 4. Evidence: it works

`evidence/delivery.txt` shows tunnelled packets arriving at h2 and h3 with
`hop_limit=0` and TTL 64. It also shows a successful plain ping to h3 with
TTL 62, so IPv4 forwarding remains unchanged.

`evidence/loop-after.txt` shows the modified loop stopping after s1 processed
2 packets and s2 processed 1 packet. The packet was forwarded by s1 with its
limit reduced from 2 to 1, forwarded by s2 with its limit reduced to 0, then
returned to s1 and dropped. This matches the design exactly.

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

## AI use

I used GitHub Copilot in VS Code to inspect the starter P4 and Python files,
explain the BMv2 log and table behavior, draft the hop-limit changes, and help
organize the report. I reviewed the changes, ran the P4 compiler and Python
syntax checks, and tested the behavior in Mininet with student ID 4903. The
evidence files contain the outputs from my own runs.

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
