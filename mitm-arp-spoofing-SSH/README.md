# MITM vs. Encryption — SSH Capture Lab

> An active man-in-the-middle attack that **succeeds at the network layer and still fails at its goal**, because the target traffic is encrypted. The companion to a plaintext HTTP MITM: same attack, opposite outcome.

![Status](https://img.shields.io/badge/lab-complete-success)
![Focus](https://img.shields.io/badge/focus-encryption%20%7C%20SSH%20%7C%20MITM-blue)
![Tools](https://img.shields.io/badge/tools-Kali%20%7C%20Wireshark%20%7C%20arpspoof-informational)

---

## Scenario

An employee opens an **SSH** session to an internal server. An attacker already has a foothold on the LAN and performs ARP cache poisoning to sit in the middle of the connection, capturing all traffic with Wireshark.

The question the lab answers: *if the attacker is genuinely in the middle, why can't they read the password?*

---

## Topology

| Host    | Role     | IP               | MAC                 |
|---------|----------|------------------|---------------------|
| Alpine  | Server (OpenSSH) | `192.168.100.10` | `00:0c:29:d3:d6:72` |
| Lubuntu | Victim   | `192.168.100.20` | `00:0c:29:96:46:11` |
| Kali    | Attacker | `192.168.100.30` | `00:0c:29:bc:e2:2b` |

All three sit on an isolated VMware LAN segment with **no route to the internet** — attack traffic never leaves the lab.

---

## Attack

The man-in-the-middle position is established exactly as in a plaintext MITM: enable IP forwarding, then poison both directions so the victim ⇄ server traffic flows through the attacker.

```bash
# On the attacker (Kali) — allow forwarding so the victim keeps connectivity
echo 1 > /proc/sys/net/ipv4/ip_forward

# Poison both directions (-t victim, -r server)
arpspoof -i eth0 -t 192.168.100.20 -r 192.168.100.10
```

`arpspoof` floods the victim with `arp reply 192.168.100.10 is-at 00:0c:29:bc:e2:2b` and the server with the mirror lie — both sides now believe the attacker is the other party.

![arpspoof poisoning both directions](screenshots/01-arpspoof.png)

Wireshark on the attacker confirms the continuous ARP reply flood — the MITM is live:

![ARP flood in Wireshark](screenshots/02-arp-flood.png)

---

## Result

The victim's SSH session connects and works normally — it is routed **through the attacker**, so the MITM genuinely succeeded:

![Victim SSH login succeeds](screenshots/03-ssh-login.png)

But filtering the captured stream (`tcp.stream eq 1`) shows only **encrypted SSHv2** — protocol banners, key-exchange negotiation, Diffie-Hellman, then ciphertext. **No password anywhere:**

![Encrypted SSHv2 in Wireshark](screenshots/04-ssh-encrypted.png)

---

## Why the attacker sees nothing

SSH negotiates an encrypted channel **above TCP, before any credentials are sent**:

1. **TCP handshake** — visible (SYN / SYN-ACK / ACK).
2. **Protocol & cipher negotiation** — both sides agree on algorithms.
3. **Diffie-Hellman key exchange** — the clever part. The exchange itself happens *in the clear* (you can see the `Diffie-Hellman Key Exchange` packets in the capture), yet the attacker **cannot derive the shared secret** from what crosses the wire. Two parties agree on a key over an open channel without ever transmitting it.
4. **Everything after** — including the password — travels inside the encrypted tunnel.

Plaintext HTTP has no steps 2–3, which is why the same attack leaks a password over HTTP but not over SSH.

---

## HTTP vs. SSH — same attack, two outcomes

|                               | HTTP (plaintext)        | SSH (encrypted)      |
|-------------------------------|-------------------------|----------------------|
| ARP spoofing succeeded        | ✅ yes                  | ✅ yes               |
| Traffic routed through attacker | ✅ yes                | ✅ yes               |
| Credentials readable in capture | ✅ `user=...&password=...` | ❌ ciphertext only |
| Lesson                        | no encryption = exposed | encryption defeats passive MITM |

---

## Detection

- A flood of gratuitous ARP replies on the segment (one host answering for many IPs).
- Duplicate MAC address appearing for two different IPs in the ARP table.
- ICMP host-redirect storm from the forwarding host (a side effect of an active forwarding MITM).

---

## Important nuance — what this does *not* prove

This demonstrates confidentiality against a **passive** sniffer who only forwards and reads. It does **not** mean SSH is immune to an **active** MITM:

- An attacker who actively terminates the SSH connection and presents a **fake host key** would attempt a true SSH interception.
- SSH defends against this with **host-key verification** — on such an attempt the client shows `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED`.
- In this lab the victim connected on first use (trust-on-first-use), so no warning appeared — the attacker was only forwarding, not impersonating the host.

> **Takeaway:** encryption protects the *contents*; host-key verification protects *who you are talking to*. Both matter.

---

## MITRE ATT&CK

| Technique | ID |
|-----------|----|
| Adversary-in-the-Middle: ARP Cache Poisoning | T1557.002 |
| Network Sniffing | T1040 |

---

## Lab environment

- **Hypervisor:** VMware Workstation, isolated LAN segment (no NAT / no internet)
- **Attacker:** Kali Linux — `arpspoof` (dsniff), Wireshark
- **Server:** Alpine Linux — OpenSSH
- **Victim:** Lubuntu

*All hosts, addresses and credentials are lab-only and fictional.*
