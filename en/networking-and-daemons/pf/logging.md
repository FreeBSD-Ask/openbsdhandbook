# PF Logging

## Introduction

When a packet is logged by PF, a copy of the packet header is sent to a pflog interface along with some additional data such as the interface the packet was transiting, the action that PF took (pass or block), etc. The pflog interface allows user-space applications to receive PF’s logging data from the kernel. If PF is enabled when the system is booted, the pflogd daemon is started. By default, pflogd listens on the ‘pflog0’ interface and writes all logged data to the ‘/var/log/pflog’ file.

## Logging Packets

In order to log packets passing through PF, the ’log’ keyword must be used. The ’log’ keyword causes all packets that match the rule to be logged. In the case where the rule is [creating state](filter.md#keeping-state)
, only the first packet seen (the one that causes the state to be created) will be logged.

The options that can be given to the ’log’ keyword are:

***all***

Causes all matching packets, not just the initial packet, to be logged. Useful for rules that create state.

**to** ***pflogN***

Causes all matching packets to be logged to the specified pflog interface. For example, when using [spamlogd(8)](https://man.openbsdhandbook.com/spamlogd.8/)
, all SMTP traffic can be logged to a dedicated pflog interface by PF. The spamlogd daemon can then be told to listen on that interface. This keeps the main PF logfile clean of SMTP traffic which otherwise would not need to be logged. Use ifconfig to create pflog interfaces. The default log interface ‘pflog0’ is created automatically.

***user***

Causes the user id and group id that owns the socket that the packet is sourced from/destined to (whichever socket is local) to be logged along with the standard log information.

Options are given in parenthesis after the ’log’ keyword; multiple options can be separated by a comma or space.

```pf
pass in log (all, to pflog1) on egress inet proto tcp to egress port 22
```

## Reading a Log File

The log file written by pflogd is in binary format and cannot be read using a text editor. [tcpdump(8)](https://man.openbsdhandbook.com/tcpdump.8/)
must be used instead.

To view the log file:

```
# tcpdump -n -e -ttt -r /var/log/pflog
```

Note that using tcpdump(8) to watch the pflog file does *not* give a real-time display. A real-time display of logged packets is achieved by using the ‘pflog0’ interface:

```
# tcpdump -n -e -ttt -i pflog0
```

**NOTE:** When examining the logs, special care should be taken with tcpdump’s verbose protocol decoding (activated via the ‘-v’ command line option). tcpdump’s protocol decoders do not have a perfect security history. At least in theory, a delayed attack could be possible via the partial packet payloads recorded by the logging device. It is recommended practice to move the log files off of the firewall machine before examining them in this way.

Additional care should also be taken to secure access to the logs. By default, pflogd will record 160 bytes of the packet in the log file. Access to the logs could provide partial access to sensitive packet payloads.

## Filtering Log Output

Because pflogd logs in tcpdump binary format, the full range of tcpdump features can be used when reviewing the logs. For example, to only see packets that match a certain port:

```
# tcpdump -n -e -ttt -r /var/log/pflog port 80
```

This can be further refined by limiting the display of packets to a certain host and port combination:

```
# tcpdump -n -e -ttt -r /var/log/pflog port 80 and host 192.168.1.3
```

The same idea can be applied when reading from the ‘pflog0’ interface:

```
# tcpdump -n -e -ttt -i pflog0 host 192.168.4.2
```

Note that this has no impact on which packets are logged to the pflogd log file; the above commands only display packets as they are being logged.

In addition to using the standard [tcpdump](https://man.openbsdhandbook.com/tcpdump.8/)
filter rules, the tcpdump filter language has been extended for reading pflogd output:

- ‘ip’ - address family is IPv4.
- ‘ip6’ - address family is IPv6.
- ‘on *int*’ - packet passed through the interface *int*.
- ‘ifname *int*’ - same as ‘on *int*’.
- ‘ruleset *name*’ - the [ruleset/anchor](anchors.md)
  that the packet was matched in.
- ‘rulenum *num*’ - the filter rule that the packet matched was rule number *num*.
- ‘action *act*’ - the action taken on the packet. Possible actions are ‘pass’ and ‘block’.
- ‘reason *res*’ - the reason that ‘action’ was taken. Possible reasons are ‘match’, ‘bad-offset’, ‘fragment’, ‘short’, ’normalize’, ‘memory’, ‘bad-timestamp’, ‘congestion’, ‘ip-option’, ‘proto-cksum’, ‘state-mismatch’, ‘state-insert’, ‘state-limit’, ‘src-limit’ and ‘synproxy’.
- ‘inbound’ - packet was inbound.
- ‘outbound’ - packet was outbound.

Example:

```
# tcpdump -n -e -ttt -i pflog0 inbound and action block and on wi0
```

This display the log, in real-time, of inbound packets that were blocked on the wi0 interface.
