# PF Forwarding

## Introduction

When you have NAT running in your office, you have the entire internet available to all your machines. What if you have a machine behind the NAT gateway that needs to be accessed from outside? This is where redirection comes in. Redirection allows incoming traffic to be sent to a machine behind the NAT gateway.

Let’s look at an example:

```pf
pass in on egress proto tcp from any to any port 80 rdr-to 192.168.1.20
```

This line redirects TCP port 80 (web server) traffic to a machine inside the network at 192.168.1.20. So, even though 192.168.1.20 is behind your gateway and inside your network, the outside world can access it.

The ‘from any to any’ part of the above ‘rdr’ line can be quite useful. If you know what addresses or subnets are supposed to have access to the web server at port 80, you can restrict them here:

```pf
pass in on egress proto tcp from 203.0.113.0/24 to any port 80 rdr-to 192.168.1.20
```

This will redirect only the specified subnet. Note this implies you can redirect different incoming hosts to different machines behind the gateway. This can be quite useful as well. For example, you could have users at remote sites access their own desktop computers using the same port and IP address on the gateway as long as you know the IP address they will be connecting from.

```pf
pass in on egress proto tcp from 203.0.113.14 to any port 80 rdr-to 192.168.1.20
pass in on egress proto tcp from 198.51.100.89 to any port 80 rdr-to 192.168.1.22
pass in on egress proto tcp from 198.51.100.178 to any port 80 rdr-to 192.168.1.23
```

A range of ports can also be redirected within the same rule:

```pf
pass in on egress proto tcp from any to any port 5000:5500 \
   rdr-to 192.168.1.20
pass in on egress proto tcp from any to any port 5000:5500 \
   rdr-to 192.168.1.20 port 6000
pass in on egress proto tcp from any to any port 5000:5500 \
   rdr-to 192.168.1.20 port 7000:*
```

These examples show ports 5000 to 5500 inclusive being redirected to 192.168.1.20.

- In rule #1, port 5000 is redirected to 5000, 5001 to 5001, etc.
- In rule #2, the entire port range is redirected to port 6000.
- And in rule #3, port 5000 is redirected to 7000, 5001 to 7001, etc.

## Security Implications

Redirection does have security implications. Punching a hole in the firewall to allow traffic into the internal, protected network potentially opens up the internal machine to compromise. If traffic is forwarded to an internal web server and a vulnerability is discovered in the web server daemon, that machine can be compromised from an intruder on the internet. From there, the intruder has a doorway to the internal network, one that is permitted to pass right through the firewall.

These risks can be minimized by keeping the externally accessed system tightly confined on a separate network. This network is often referred to as a Demilitarized Zone (DMZ) or a Private Service Network (PSN). This way, if the web server is compromised, the effects can be limited to the DMZ/PSN network by careful filtering of the traffic permitted to and from the DMZ/PSN.

## Redirection and Reflection

Often, redirection rules are used to forward incoming connections from the internet to a local server with a private address in the internal network or LAN, as in:

```pf
server = 192.168.1.40
pass in on egress proto tcp from any to egress port 80 rdr-to $server port 80
```

But when the redirection rule is tested from a client on the LAN, it doesn’t work. The reason is that redirection rules apply only to packets that pass through the specified interface (egress, the external interface, in the example). Connecting to the external address of the firewall from a host on the LAN, however, does not mean the packets will actually pass through its external interface. The TCP/IP stack on the firewall compares the destination address of incoming packets with its own addresses and aliases and detects connections to itself as soon as they have passed the internal interface. Such packets do not physically pass through the external interface, and the stack does not simulate such a passage in any way. Thus, PF never sees these packets on the external interface, and the redirection rule, specifying the external interface, does not apply.

Adding a second redirection rule for the internal interface does not have the desired effect either. When the local client connects to the external address of the firewall, the initial packet of the TCP handshake reaches the firewall through the internal interface. The redirection rule does apply and the destination address gets replaced with that of the internal server. The packet gets forwarded back through the internal interface and reaches the internal server. But the source address has not been translated, and still contains the local client’s address, so the server sends its replies directly to the client. The firewall never sees the reply and has no chance to properly reverse the translation. The client receives a reply from a source it never expected and drops it. The TCP handshake then fails and no connection can be established.

Still, it’s often desirable for clients on the LAN to connect to the same internal server as external clients and to do so transparently. There are several solutions for this problem:

### Split-horizon DNS

It’s possible to configure DNS servers to answer queries from local hosts differently than external queries so that local clients will receive the internal server’s address during name resolution. They will then connect directly to the local server, and the firewall isn’t involved at all. This reduces local traffic since packets don’t have to be sent through the firewall.

### Moving the Server into a Separate Local Network

Adding an additional network interface to the firewall and moving the local server from the client’s network into a dedicated network (DMZ) allows redirecting of connections from local clients in the same way as the redirection of external connections. Use of separate networks has several advantages, including improving security by isolating the server from the remaining local hosts. Should the server (which in our case is reachable from the Internet) ever become compromised, it can’t access other local hosts directly as all connections have to pass through the firewall.

### TCP Proxying

A generic TCP proxy can be setup on the firewall, either listening on the port to be forwarded or getting connections on the internal interface redirected to the port it’s listening on. When a local client connects to the firewall, the proxy accepts the connection, establishes a second connection to the internal server, and forwards data between those two connections.

### RDR-TO and NAT-TO Combination

With an additional NAT rule on the internal interface, the lacking source address translation described above can be achieved.

```pf
pass in on $int_if proto tcp from $int_net to egress port 80 rdr-to $server
pass out on $int_if proto tcp to $server port 80 received-on $int_if nat-to $int_if
```

This will cause the initial packet from the client to be translated again when it’s forwarded back through the internal interface, replacing the client’s source address with the firewall’s internal address. The internal server will reply back to the firewall, which can reverse both NAT and RDR translations when forwarding to the local client. This construct is rather complex as it creates two separate states for each reflected connection. Care must be taken to prevent the NAT rule from applying to other traffic, for instance connections originating from external hosts (through other redirections) or the firewall itself. Note that the ‘rdr-to’ rule above will cause the TCP/IP stack to see packets arriving on the internal interface with a destination address inside the internal network.
