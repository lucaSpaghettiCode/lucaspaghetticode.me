## Context

This work comes out of an R&D internship where the daily reality was industrial devices that are small in every dimension that matters: ESP32-class microcontrollers with a few hundred kilobytes of RAM, intermittent connectivity over NAT-ed cellular or factory Wi-Fi, and battery or power budgets that make every transmitted byte a line item. The firmware side ran on a Python runtime on top of FreeRTOS — which explains the otherwise odd pairing in this project's tech chips: **Python on the devices, Go in the backend**.

The architectural question was: how do you build a *zero-trust* pipeline — every hop authenticated, every identity verified, no segment of the network assumed safe — when one end of the pipeline can't afford a TLS certificate chain handshake, let alone a service mesh sidecar? The answer was a deliberately asymmetric design: **CoAP over DTLS** southbound where constraints rule, **MQTT over TLS** northbound where infrastructure rules, and a **Raspberry Pi 4 gateway** as the explicit, auditable trust boundary between the two worlds.

---

## Southbound: CoAP, Because the Constraints Are Real

**CoAP (RFC 7252)** is what you get if you ask "what would HTTP look like if every byte cost something": a 4-byte fixed header over UDP, REST semantics (GET/PUT/POST/DELETE on resources), and reliability as an opt-in per message — *confirmable* messages get ACKed and retransmitted with exponential back-off; *non-confirmable* ones don't, which is exactly right for telemetry where the next reading supersedes the lost one.

Two extensions did the heavy lifting:

- **Observe (RFC 7641)** inverts the polling model: the gateway registers interest in a resource once, and the device pushes notifications when the value changes. For a sensor reporting on change rather than on schedule, this collapsed both radio-on time and message volume.
- **Block-wise transfer (RFC 7959)** carries anything larger than a single datagram — most critically OTA firmware images, chunked into 1024-byte blocks, each independently ACKed and resumable. Re-sending one lost block instead of restarting a multi-megabyte transfer over flaky cellular is the difference between OTA that works and OTA that's a support ticket.

Payloads are **CBOR (RFC 8949)**, not JSON. On our typical telemetry frame the encoded size dropped by roughly 60%, and just as importantly the encoder is trivially cheap on a microcontroller — no string formatting, no escaping.

Security southbound is **DTLS 1.2 in the RFC 7925 IoT profile**: pre-shared keys with `TLS_PSK_WITH_AES_128_CCM_8`. PSK over raw public keys or X.509 was a deliberate trade — the handshake is two flights with no certificate parsing, no chain validation, no clock dependency (a real issue: a device that boots with no valid time can't validate certificate expiry). Each device gets a **unique per-device PSK provisioned at manufacture**, so compromising one device burns one key, not a fleet. The persistent operational headache was NAT rebinding: a UDP "connection" that goes quiet loses its NAT mapping, the source port changes, and the DTLS session no longer matches. **Connection ID (RFC 9146** — published, conveniently, the same year**)** solves precisely this by decoupling the session from the 5-tuple; where library support wasn't there yet, the fallback was tuned session resumption, which still beats a full handshake.

---

## The Gateway: An Explicit Trust Boundary

The **Raspberry Pi 4** gateway is where zero-trust stops being a slogan and becomes a data structure. It terminates DTLS, validates the device identity against its provisioned key, and **re-publishes northbound with the device identity attached as a verified claim** — the upstream never has to take the gateway's word for *which* device sent a message without evidence of *how* the gateway knows.

Protocol translation maps CoAP semantics onto MQTT topics:

```
coap://gw/telemetry/{device}   →  site/{site}/dev/{device}/telemetry   (QoS 1)
coap://gw/state/{device}       →  site/{site}/dev/{device}/state       (retained)
command topic (subscribe)      →  CON message pushed to device
```

Retained messages carry last-known state so a reconnecting consumer doesn't wait for the next report; the gateway's **MQTT Last Will** marks the entire site unreachable if the gateway itself drops, which is operationally distinct from — and more urgent than — any single device going quiet. The gateway holds its own X.509 client certificate for mutual TLS to the broker: identity on *both* ends of the northbound link, no password auth anywhere.

The gateway buffers to local flash when the uplink is down (store-and-forward with QoS 1 redelivery), which sounds like a detail until a factory's uplink dies for six hours and no telemetry is lost.

---

## Northbound: Go Microservices, Zero-Trust Between Services Too

The backend is a small constellation of **Go services** — chosen for static binaries, a runtime measured in megabytes, and a concurrency model that maps naturally onto "thousands of slow, mostly-idle message streams":

- **device-registry** — the source of truth for identity: device → keys, site, firmware version, authorization scope. Broker ACLs are *generated* from the registry, so a device can publish only to its own topic subtree; a compromised device cannot even subscribe to its neighbour's data.
- **ingest** — consumes telemetry off the broker, validates against per-device schemas, writes to time-series storage.
- **command** — the only service authorized to publish downstream commands, with an audit log of every command and its originating principal.
- **ota** — firmware rollout orchestration: staged cohorts, version pinning, and the block-wise transfer bookkeeping on the gateway side.

Zero-trust *inside* the backend meant refusing the comfortable assumption that "it's on our network" implies "it's allowed": **mTLS on every service-to-service connection** with short-lived certificates issued by an internal CA (SPIFFE-style identities baked into the SAN), and authorization checked per-RPC against the service identity, not the network location. No service trusts the broker's payloads either — ingest re-validates that the claimed device identity matches the topic it arrived on, because the broker is infrastructure, and infrastructure gets compromised. We tracked the then-brand-new **ACE-OAuth framework (RFC 9200)** as the standards-track future for pushing token-based authorization all the way down to the constrained devices; in 2022 the library ecosystem wasn't there, and the per-device PSK + registry-derived ACL model was the engineering-honest version of the same idea.

---

## Python's Other Job: Proving It Works

Beyond the device firmware, Python carried the verification load. An **aiocoap-based device simulator** could impersonate hundreds of devices — same DTLS handshake, same CBOR payloads, same Observe registrations — which made two things possible that hardware alone never would have: **load testing** (500 simulated devices against one Pi 4 gateway: DTLS session memory, not CPU, is what you run out of first) and a **pytest harness** running the full path — simulated device → gateway → broker → ingest — as an integration suite in CI, including the unhappy paths: expired sessions, NAT rebinding mid-transfer, a device presenting a valid key for the wrong identity.

The lasting lesson wasn't any single protocol choice. It was that *zero-trust in IoT is an exercise in identity bookkeeping under constraint* — the crypto is the easy part; the hard part is making sure that at every hop, from a 240 KB-of-RAM sensor to a Go service three networks away, the question "who exactly is saying this, and how do we know?" has an answer that doesn't involve trusting the wire.
