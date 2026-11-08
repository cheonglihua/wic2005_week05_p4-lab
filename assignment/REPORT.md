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
switch to 1,594 on s1 and 1,595 on s2 during the observation period,
confirming that the loop continued without stopping.

The two recorded count outputs were:

     logs/s1.txt:511
     logs/s2.txt:511
     logs/s1.txt:1594
     logs/s2.txt:1595

The `ipv4_forward` action decrements the IPv4 TTL, but tunnelled packets use
`myTunnel_forward`, which does not modify the IPv4 header. Therefore, the
IPv4 TTL remains at 64 and cannot stop the tunnel loop. Even for normal IPv4
forwarding, the supplied P4 program does not explicitly check whether the TTL
has reached zero. Since the TTL is an unsigned 8-bit field, decrementing it
from zero causes it to wrap around to 255. As a result, the existing IPv4
forwarding logic does not reliably prevent forwarding loops either.

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

For example, the receiver reported:

     Received (tunnelled, dst_id=2, hop_limit=0) from 10.3.0.1 to 10.3.0.2, TTL 64: 'to h2'

The post-change loop count stopped at:

     logs/s1.txt:2
     logs/s2.txt:1

`evidence/loop-after.txt` shows the modified loop stopping after s1 processed
2 packets and s2 processed 1 packet. The packet was forwarded by s1 with its
limit reduced from 2 to 1, forwarded by s2 with its limit reduced to 0, then
returned to s1 and dropped. This matches the design exactly.

## 5. Evaluation

My prediction before the mixed rollout was that old s2 would read
`proto_id=0x0800` and `dst_id=2` correctly, but interpret the new hop-limit
byte as the first byte of IPv4. That prediction was correct. The packet was
forwarded toward h2, but h2 did not decode it as UDP. The capture in
`evidence/mixed.txt` shows old s2's four-byte tunnel header `0800 0002`, then
the extra byte `01`, followed by the IPv4 bytes beginning `45 00`. Old s2
therefore forwarded using the right tunnel table key while reconstructing a
malformed payload.

The captured bytes included this exact line:

     0x0010:  0002 0145 0000 2100 0100 0040 1366 c20a

For the mixed loop, s1 processed 3 packets and old s2 processed 2 before the
packet returned to s1 with a zero value in the byte that the new program
interpreted as `hop_limit`. New s1 decremented it, while old s2 did not. The
loop stopped in this particular run, but only because the byte layout happened
to preserve a usable value; mixed versions do not provide a reliable protocol
contract.

A safe rollout option is a new EtherType for the new format. Old switches then
do not parse the new packet as the old header, and the control plane can avoid
installing new-format traffic on old paths. A version field is another option:
both versions can share an EtherType, but every switch must inspect the version
before choosing the correct header length and behavior. A new EtherType is
simpler to fail closed; a version field is more flexible but requires old
switches to be explicitly programmed to recognize and safely reject or pass
the new version. In either case, deploy parser support first, then enable new
packets only after all paths support them.

The data-plane hop limit protects individual packets immediately: it can stop
a forwarding loop even when the control plane is unaware of the loop. It
cannot prove that the configured network topology is loop-free or repair bad
table entries. Week 3's control-plane `check_paths()` can detect path or
configuration loops before traffic is sent and can reject an unsafe topology,
but it cannot catch a transient runtime table change, parser mismatch, or a
loop created after the check. Real networks use both because configuration
validation and per-packet containment cover different failure modes.

On a line-rate hardware switch, the change adds an 8-bit header field, one
byte of parser/deparser bandwidth, and a small compare, decrement, and
conditional drop in the ingress pipeline. It may consume a little parser,
match-action, and metadata resource, but it does not require a new table key.
The main hardware concern is fitting the extra logic and header width into the
target's fixed pipeline stages; if it fits, the per-packet cost is constant
and should not reduce throughput, though it consumes some scarce resources.

## AI use

**AI usage level: Level 3 - AI-Assisted.** I used GitHub Copilot in VS Code to inspect the starter P4 and Python files,
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
