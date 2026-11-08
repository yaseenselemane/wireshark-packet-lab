# Wireshark packet lab

Seven small captures from my own laptop on my home WiFi, each one triggered by a single command, and what the packets actually show. I captured with **tshark** (Wireshark's command-line version) so every capture is filtered to just the traffic I caused.

Everything here is real output. MAC addresses are replaced with `laptop-MAC` / `router-MAC`. The decoded packets are in [`captures/`](captures/).

Part of my networking labs: [ping and traceroute](https://github.com/yaseenselemane/ping-and-traceroute) · [nmap](https://github.com/yaseenselemane/nmap-field-notes) · [WiFi survey](https://github.com/yaseenselemane/wifi-survey) · **Wireshark** · [Packet Tracer](https://github.com/yaseenselemane/cisco-packet-tracer-labs) · [CCTV network](https://github.com/yaseenselemane/cctv-network-lab) · [subnet calculator](https://github.com/yaseenselemane/subnet-calculator) · [vantage](https://github.com/yaseenselemane/vantage)

## How each capture was made

Start a capture with a **capture filter** (so nothing else gets recorded), run one command, stop:

```bash
tshark -i Wi-Fi -a duration:8 -f "arp" -w 02-arp.pcapng     # capture for 8 s, ARP only
ping -n 1 192.168.0.139                                       # ...while this runs
tshark -r 02-arp.pcapng                                       # read it back
```

| # | Capture filter | Triggered by |
|---|---|---|
| 1 | `icmp and host 1.1.1.1` | `ping -n 2 1.1.1.1` |
| 2 | `arp` | `ping` a device that isn't in the ARP cache |
| 3 | `udp port 53` | `nslookup example.com`, `nslookup github.com 1.1.1.1` |
| 4 | `icmp` | `tracert -d -h 6 1.1.1.1` |
| 5 | `udp port 67 or udp port 68` | `ipconfig /renew` |
| 6 | `tcp port 80 and host 104.20.23.154` | `curl http://example.com` |
| 7 | `tcp port 443 and host 104.20.23.154` | `curl https://example.com` |

---

## 1. Ping: request, reply, TTL

```
1  192.168.0.162 → 1.1.1.1        Echo (ping) request   ttl=128
2  1.1.1.1       → 192.168.0.162  Echo (ping) reply     ttl=58     +25 ms
```

The request leaves with TTL 128 (Windows) and the reply arrives with 58 (Cloudflare started at 64, so about 6 routers back). Round trip: 25 ms.

## 2. ARP: "who has this IP?"

```
1  laptop-MAC → Broadcast  ARP Who has 192.168.0.139? Tell 192.168.0.162
3  laptop-MAC → Broadcast  ARP Who has 192.168.0.139? Tell 192.168.0.162
6  laptop-MAC → Broadcast  ARP Who has 192.168.0.139? Tell 192.168.0.162
2  router-MAC → Broadcast  ARP Who has 192.168.0.231? Tell 192.168.0.1
4  router-MAC → Broadcast  ARP Who has 192.168.0.130? Tell 192.168.0.1
```

Before a PC can send anything on the LAN it needs the target's **MAC address**, so it shouts to everyone (broadcast): "who has 192.168.0.139?". Here **nobody answered**: three tries, one second apart. The ping output looked like this:

```
Reply from 192.168.0.162: Destination host unreachable.
    Packets: Sent = 1, Received = 1, Lost = 0 (0% loss)
```

That "reply" came from **my own laptop**, saying it couldn't find the device, and Windows counts it as 0% loss. The capture shows the truth: the ping never left the laptop, because ARP failed first. In the background, the router was also asking for `.231` and `.130`, devices that have left the network.

## 3. DNS: turning a name into an IP

```
3  192.168.0.162 → 192.168.0.1  Standard query A example.com.lan
4  192.168.0.1   → 192.168.0.162 response No such name
7  192.168.0.162 → 192.168.0.1  Standard query A example.com
8  192.168.0.1   → 192.168.0.162 response A 104.20.23.154 A 172.66.147.243     +25 ms
9  192.168.0.162 → 192.168.0.1  Standard query AAAA example.com              (no answer)
```

- Windows first tried **`example.com.lan`**: the router hands out the DNS suffix `lan` (see the DHCP capture), so the PC tries that first and only then the real name. That's two wasted lookups on every new name.
- One name returned **two IPs**: big sites spread load across several servers.
- The **AAAA** (IPv6) query got no answer. This network has no IPv6.
- Asking Cloudflare directly (`nslookup github.com 1.1.1.1`) worked the same way, and showed the server's own name: `one.one.one.one`.

## 4. Traceroute: TTL tricks

```
1  192.168.0.162 → 1.1.1.1         Echo request  ttl=1
2  192.168.0.1   → 192.168.0.162   Time-to-live exceeded          ← hop 1: my router
7  192.168.0.162 → 1.1.1.1         Echo request  ttl=2            ← hop 2: no reply at all
10 192.168.0.162 → 1.1.1.1         Echo request  ttl=3
11 10.11.113.1   → 192.168.0.162   Time-to-live exceeded          ← hop 3: inside the ISP (private 10.x = CGNAT)
15 192.168.0.162 → 1.1.1.1         Echo request  ttl=4
17 41.216.207.112 → 192.168.0.162  Time-to-live exceeded          ← hop 4: ISP backbone
```

This is what `tracert` really does: send pings with TTL 1, 2, 3… and each router that drops one sends back "time exceeded", revealing itself. Hop 2 sends nothing, which is the `* * *` in the tracert output.

## 5. DHCP: renewing the lease

```
1  192.168.0.162 → 192.168.0.1   DHCP Request   Transaction ID 0x82b88612
2  192.168.0.1   → 192.168.0.162 DHCP ACK       Transaction ID 0x82b88612     +8 ms
```

What the ACK contains ([full decode](captures/05-dhcp-full-decode.txt)):

| Option | Value |
|---|---|
| Your IP | 192.168.0.162 /24 |
| Router (gateway) | 192.168.0.1 |
| DNS server | 192.168.0.1 |
| Domain name | `lan` |
| Lease time | 12 hours |
| Renew (T1) / Rebind (T2) | 5 h 27 m / 9 h 57 m |

A brand-new device does all four steps: **D**iscover, **O**ffer, **R**equest, **A**ck. A device that already has an address only renews, so it skips straight to **Request → ACK** (sent to the router directly, not broadcast). The matching Transaction ID is how the two packets are paired up.

## 6. HTTP: everything in plain text

```
1  192.168.0.162 → 104.20.23.154  TCP 55320 → 80 [SYN]
2  104.20.23.154 → 192.168.0.162  TCP 80 → 55320 [SYN, ACK]       +18 ms
3  192.168.0.162 → 104.20.23.154  TCP 55320 → 80 [ACK]            ← 3-way handshake done
4  192.168.0.162 → 104.20.23.154  HTTP GET / HTTP/1.1
7  104.20.23.154 → 192.168.0.162  HTTP HTTP/1.1 200 OK (text/html)
9  192.168.0.162 → 104.20.23.154  TCP [FIN, ACK]                  ← polite close
```

- **The 3-way handshake:** SYN ("can we talk?"), SYN-ACK ("yes"), ACK ("great"). Every TCP connection starts this way.
- Because it's plain HTTP, **anyone on the path can read everything**: the request, the headers and the whole page ([headers](captures/06-http-headers.txt)).
- The headers show `Server: cloudflare`, `cf-cache-status: HIT` and a CF-RAY ending in **`JNB`**: the page came from Cloudflare's cache in **Johannesburg**, not from the site's own server. That's why it took 18 ms, not 200.
- **The ports:** the server uses 80, and my side used a random high port (55320). That's how many connections share one IP.

## 7. HTTPS: what encryption hides (and doesn't)

```
1  192.168.0.162 → 104.20.23.154  TCP 55346 → 443 [SYN]
3  192.168.0.162 → 104.20.23.154  TCP 55346 → 443 [ACK]
4  192.168.0.162 → 104.20.23.154  TLSv1.2 Client Hello (SNI=example.com)
6  104.20.23.154 → 192.168.0.162  TLSv1.3 Server Hello, Change Cipher Spec
9  104.20.23.154 → 192.168.0.162  TLSv1.3 Application Data           ← encrypted from here on
```

- Same TCP handshake, then a **TLS handshake** on top.
- The **Client Hello is readable**, and it carries **SNI = example.com**. So even with HTTPS, someone watching the network can see **which site** you visit, just not what you do there ([decode](captures/07-tls-client-hello-decode.txt)).
- The server picked **TLS 1.3** with **AES-256-GCM**. The Client Hello is labelled "TLSv1.2" only for compatibility: the real version is negotiated in the "supported versions" extension (it offered 1.3 and 1.2).
- After that, every packet is just "Application Data". The page, the headers and even the URL path are hidden.

---

## What I learned

1. **Capture filters keep you sane.** `-f "udp port 53"` records only DNS, so a 7-second capture is 19 packets instead of thousands, and nothing private from other apps gets recorded.
2. **ARP comes before everything on a LAN.** No MAC, no packet. A failed ARP looks like "Destination host unreachable" from *your own IP*, with "0% loss", which is misleading without a capture.
3. **DNS suffixes cause hidden extra lookups.** Every new name was tried as `name.lan` first, because DHCP handed out the `lan` domain.
4. **DHCP renewal is two packets, not four.** DORA is only for new devices. Renewals go straight to Request and ACK, matched by the Transaction ID, and the lease timers (T1 and T2) decide when that happens.
5. **Traceroute is just TTL plus "time exceeded" replies**, and silent hops are routers that don't send them.
6. **The TCP 3-way handshake (SYN, SYN-ACK, ACK) starts every connection**, and FIN/ACK closes it cleanly.
7. **HTTP is a postcard, HTTPS is a sealed envelope with the address on the outside.** The SNI in the Client Hello still shows which site you're visiting.
8. **A CDN changes the path.** The CF-RAY "JNB" header showed the page came from a Johannesburg cache, which explains the 18 ms response.
9. **Read the real version, not the label.** The Client Hello said TLS 1.2. The supported-versions extension and the Server Hello showed TLS 1.3.
