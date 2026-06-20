# ADR-002 — One-Way OT→IT Edge Bridge

- **Status:** Accepted
- **Context area:** OT/ICS security, network segmentation, edge ingestion

## Context

The platform needs live-ish data from the plant floor: fault/sensor states from PLCs and the
contents of an intranet document share. But the OT network runs industrial control systems
where the cost of a mistake is measured in safety and production, not error logs. Plant and
OT-security stakeholders set a non-negotiable requirement:

> Nothing on the IT/platform side may initiate a connection into the OT network, and nothing
> may write to a controller.

The naive integration — give the platform credentials and let it reach into OT to read tags
and files — fails this requirement outright. It creates an inbound attack path into OT and a
code path that *could* write to a controller, both of which are unacceptable regardless of how
carefully they're guarded.

## Decision

Introduce a dedicated **edge gateway node** as the *single* crossing point, implementing a
**poll-and-push, read-only, outbound-only** bridge:

- The gateway sits on the OT network and is the **only** device with a route both into OT and
  outbound to the platform.
- It **reads** from the PLC (connect → read one data block → disconnect; no persistent session)
  and **reads** from the document share — the edge connector performs read-only polling by
  design. No write-capable functions are linked into the poller, so it has no code path to write
  to a controller.
- It buffers locally and **pushes** data outbound to the platform over **mutual TLS on port 443
  only**, with the client certificate pinned to this gateway.
- Its host firewall **denies all inbound** and permits only that single outbound destination.
- Network segmentation ensures there is **no route from the platform back into the OT VLAN**.

The platform is, from OT's perspective, a sink: data flows out, commands never flow in.

## Consequences

**Positive**
- The design targets one-way OT→IT isolation with deny-by-default segmentation, and is meant to
  be **verifiable from configuration** — firewall rules, the pinned outbound mTLS tunnel, and the
  absence of write functions — rather than merely asserted in a policy document.
- The integration is dramatically **easier to reason about and audit** than any bidirectional
  design: there is exactly one direction and one channel to review.
- The blast radius of a platform compromise is designed to stop at the boundary: with
  deny-by-default segmentation, an attacker on the platform is intended to have no network path
  into OT.
- The gateway is a clean place to concentrate hardening (minimal OS, dedicated service account,
  credential vault, disabled removable media).

**Negative / accepted trade-offs**
- **No remote control of OT** from the platform — by design. Any future need to actuate
  equipment would require a separate, separately-reviewed mechanism; it is explicitly out of
  scope here.
- Data is **near-real-time, not hard-real-time** — it reflects the most recent poll, not an
  instantaneous controller state. For maintenance intelligence (faults, history, documents),
  this is entirely sufficient.
- A poll loop adds a tiny, bounded load on the controller; mitigated by reading a single block
  on a modest interval and disconnecting between reads.

**Mitigations / guarantees**
- The read-only posture is built in at the protocol and binary level, not left to convention.
- The pinned client certificate means a stolen network position alone cannot impersonate the
  gateway to the platform.
- An OT-side incident response begins with **physically disconnecting the gateway**, which by
  construction severs the only path in or out.
