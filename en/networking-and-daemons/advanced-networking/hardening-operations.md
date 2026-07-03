# Hardening and Operational Safety

## Synopsis

This chapter provides hardened defaults and safe operating procedures for OpenBSD network systems. It covers anti-spoofing and reverse-path filtering in [pf.conf(5)](https://man.openbsdhandbook.com/pf.conf.5/)
, disciplined rule rollouts with [pfctl(8)](https://man.openbsdhandbook.com/pfctl.8/)
, minimal-access administration with [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
, secure management-plane exposure via [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
, secret file hygiene for daemons (for example, [iked.conf(5)](https://man.openbsdhandbook.com/iked.conf.5/)
), and patching with [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
). It emphasizes predictable change, roll-back paths, and audit-ready configurations.

## Design Considerations

- **Threat model.** Protect the management plane first. Restrict administration origins, prefer keys over passwords, and default-deny all east–west traffic not required by policy.
- **Spoofing and path symmetry.** Enforce source validity with interface-based anti-spoofing and unicast RPF checks.
- **Rule safety.** Validate and stage PF changes with syntax checks and temporary roll-back plans. Maintain a failsafe anchor that preserves out-of-band access during rollouts.
- **Secrets and ownership.** Keep keys and PSKs readable by root only. Prefer daemon-provided chroots and privilege separation.
- **Patching and provenance.** Apply errata quickly. Track configuration in version control and record change intents in commits.
- **Logging for action.** Ship logs off the box; keep local rotation sane and verify clocks before correlating events.

## Configuration

### 1) Hardened PF skeleton with anti-spoofing, uRPF, and bogon tables

The example below is a conservative baseline. It assumes `em0` is WAN and `em1`/`em2` are inside interfaces. It rejects obviously forged sources and blocks packets that fail a reverse-path check on the receiving interface.

```pf
## /etc/pf.conf — hardened baseline with spoofing controls

set skip on lo

# Interfaces
wan   = "em0"
inside= "{ em1, em2 }"

# -------------------------
# Tables
# -------------------------
# Commonly spoofed IPv4 sources (adjust as needed; keep current in ops docs)
table <bogons4> persist { \
    0.0.0.0/8, 10.0.0.0/8, 100.64.0.0/10, 127.0.0.0/8, \
    169.254.0.0/16, 172.16.0.0/12, 192.0.2.0/24, 192.168.0.0/16, \
    198.18.0.0/15, 198.51.100.0/24, 203.0.113.0/24, 224.0.0.0/4, 240.0.0.0/4 }

table <bogons6> persist { \
    ::/128, ::1/128, ::ffff:0:0/96, 64:ff9b::/96, \
    100::/64, 2001:db8::/32, fc00::/7, fe80::/10, ff00::/8 }

# Inside networks (adjust to your deployment)
table <inside4> persist { 10.0.0.0/8, 192.168.0.0/16 }
table <inside6> persist { 2001:db8:10::/48 }

# -------------------------
# Normalization
# -------------------------
scrub in all

# -------------------------
# Base policy
# -------------------------
block all

# Anti-spoof on all relevant interfaces
antispoof quick for { $wan, $inside }

# Reverse-path filtering: drop traffic whose source would not route back on the rx interface
block in quick on $wan from urpf-failed
block in quick on $inside from urpf-failed

# Bogon filtering on WAN
block in quick on $wan inet from <bogons4> to any
block in quick on $wan inet6 from <bogons6> to any

# Permit inside to anywhere (refine per policy; examples below)
pass in on $inside inet  from <inside4> to any keep state
pass in on $inside inet6 from <inside6> to any keep state

# Outbound from the firewall itself
pass out on egress proto { tcp udp icmp } keep state

# Example: published HTTPS service; apply synproxy and limits
web_svc = "10.10.10.50"
rdr on $wan proto tcp from any to ($wan) port 443 -> $web_svc
pass in on $wan proto tcp to $web_svc port 443 \
    flags S/SA synproxy state (max-src-conn 50, max-src-conn-rate 30/5)
```
> Keep bogon lists accurate in your runbook. Tables are cheap to update and easy to audit during incidents.

### 2) Failsafe anchor and safe PF rollouts

Maintain a minimal, first-loaded anchor that guarantees management access from a restricted origin. Load it before any restrictive rules and keep it small.

```pf
## /etc/pf.failsafe.conf — allow SSH from a management /32 only
mgmt = "198.51.100.10"

pass in quick on egress proto tcp from $mgmt to (egress) port 22 keep state
```

Reference it at the top of your main policy:

```pf
## Excerpt — top of /etc/pf.conf
anchor "failsafe"
load anchor "failsafe" from "/etc/pf.failsafe.conf"
```

**Staged reload with validation and roll-back timer** (local shell on the router):

```sh
# install -b /etc/pf.conf
  # Keep a backup as /etc/pf.conf~ automatically
# pfctl -nf /etc/pf.conf
  # Syntax check; no load
# at now + 5 minutes <<< 'pfctl -f /etc/pf.conf~'
  # Roll-back PF to the previous ruleset in 5 minutes (requires atd(8))
# pfctl -f /etc/pf.conf
  # Load the new rules atomically
# atq
  # Confirm the rollback job is queued
# atrm <jobid>
  # Cancel rollback after you confirm access and function
```

If `at(1)` is unavailable, keep a second session open (serial/IPMI/ILO) while loading the new policy and verify end-to-end before closing the out-of-band path.

### 3) Management-plane access: SSH and doas

Restrict SSH exposure and disable weak options. Keep privileged escalation explicit and minimal.

**sshd(8) hardening** in [sshd\_config(5)](https://man.openbsdhandbook.com/sshd_config.5/)
:

```
## /etc/ssh/sshd_config — minimal hardened example
ListenAddress 10.10.10.1
AddressFamily inet
Protocol 2
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
ClientAliveInterval 30
ClientAliveCountMax 2
AllowUsers netops
# Optional: restrict source subnets with Match
# Match Address 198.51.100.0/24
#     AllowUsers netops
```

```sh
# rcctl reload sshd
  # Apply changes without disconnecting existing sessions
```

**doas(1)** policy in [doas.conf(5)](https://man.openbsdhandbook.com/doas.conf.5/)
):

```sh
## /etc/doas.conf — least privilege for routine ops
permit persist keepenv :wheel as root cmd pfctl
permit persist keepenv :wheel as root cmd rcctl
permit persist keepenv :wheel as root cmd tail args -f /var/log/messages
# Deny by default; enumerate additional commands as required
```

### 4) Secret file hygiene and daemon posture

Ensure private keys and PSKs are root-readable only. Many base daemons already chroot and drop privileges; keep defaults intact.

```sh
# chmod 600 /etc/iked.conf
  # PSKs and identities for iked(8)
# chmod 600 /etc/ssl/private/*.key
  # TLS private keys used by httpd(8) and other daemons
# chown -R root:wheel /etc/ssl/private
# rcctl check iked
  # Confirm daemon status in the current chroot/priv-sep model
```

### 5) Patching and provenance

Apply errata with [syspatch(8)](https://man.openbsdhandbook.com/syspatch.8/)
). Verify set signatures with [signify(1)](https://man.openbsdhandbook.com/signify.1/)
during upgrades. Plan maintenance windows and reboot when required by the patch notes.

```sh
# syspatch
  # Apply available binary patches; review /var/syspatch logs
# signify -C -p /etc/signify/openbsd-78-base.pub -x SHA256.sig
  # Verify set signatures during upgrades (adjust key/filenames per release notes)
```

> Use version control for `/etc` (for example, `rc.conf.local`, `pf.conf`, `vm.conf`, `rad.conf`) and include a human-readable change rationale with each commit.

## Verification

- **PF exposure and counters**:

```sh
$ pfctl -vvsr | egrep 'failsafe|urpf|antispoof|bogons'
  # Confirm presence and order of hardening rules
$ pfctl -s state | wc -l
  # State count; watch during change windows
$ pfctl -s Interfaces
  # Interface list and status as seen by PF
```

- **Tables and lookups**:

```sh
$ pfctl -t bogons4 -T show | head
  # Verify bogon table content
$ pfctl -t inside4 -T test 192.168.1.10
  # Expect "in table" for an inside address
```

- **Management-plane checks**:

```sh
$ ssh -o PreferredAuthentications=publickey netops@10.10.10.1 true
  # Non-interactive key auth succeeds
$ ssh -o PreferredAuthentications=password netops@10.10.10.1 true
  # Should fail if PasswordAuthentication no
```

- **Patch state**:

```sh
$ syspatch -l
  # List installed patches
```

## Troubleshooting

- **Locked out after PF reload.** Use your out-of-band console. Confirm that the failsafe anchor loads before restrictive rules and that the management source matches the `AllowUsers` and PF criteria. If you must recover rapidly, you can disable filtering with `pfctl -d` from the console, fix the policy, and re-enable with `pfctl -e`.
- **Legitimate traffic drops on WAN.** Inspect `urpf-failed` hits with `tcpdump -ni $wan` while correlating PF counters. Return-path failure often indicates asymmetric routing; adjust policy for those flows or remove the strict uRPF rule from the affected interface.
- **Bogon table blocks intended partners.** Documentation/demo prefixes (for example, `192.0.2.0/24`) occasionally appear in lab peering. Remove those entries temporarily or maintain a separate `<bogons4_prod>` table for production.
- **SSH disconnects during long operations.** Use `ClientAliveInterval` and keep-alives; consider running disruptive changes inside a `tmux` or `screen` session over the out-of-band console.
- **Daemon refuses to read keys.** Confirm file mode `600` and correct path. Many daemons require keys under `/etc/ssl/private` with root-only permissions. Review daemon logs in `/var/log/daemon` and service-specific files.
- **Patch requires reboot.** Some `syspatch(8)` updates patch the kernel or core libraries. Schedule a short reboot window and verify services come back under `rcctl`.

## See Also

- [Networking](../../install-and-configure/networking.md)
- [OpenBGPD](../services/networking-services/openbgpd.md)
- Related: [**High Availability and State Replication**](high-availability.md)
- Related: [**Telemetry, Logging, and Flow Export**](telemetry-logging-flow.md)
- Related: [**Troubleshooting Playbooks**](troubleshooting-playbooks.md)
