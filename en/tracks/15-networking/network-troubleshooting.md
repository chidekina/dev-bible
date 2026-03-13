# Network Troubleshooting

A systematic approach to diagnosing network issues — from "is it even reachable?" to "why is this specific HTTP request slow?".

## Troubleshooting Methodology

```
1. Identify the symptom precisely (timeout? refused? wrong data?)
2. Narrow the layer: physical → network → transport → application
3. Test connectivity at each layer
4. Isolate: is it the client, the network, or the server?
5. Confirm the fix works and understand why
```

## Essential Tools

### ping — Layer 3 Connectivity

```bash
# Basic connectivity test (ICMP echo)
ping google.com
ping 8.8.8.8

# Specific count
ping -c 4 google.com

# Check packet loss and round-trip time
ping -c 100 google.com | tail -2
```

Interpret results:
```
64 bytes from 142.250.80.46: icmp_seq=1 ttl=117 time=12.4 ms

ttl=117: started at 128 (Windows server) → 11 hops away
time=12.4ms: round-trip latency
packet loss: any loss is concerning in a LAN; some OK on internet

If ping fails:
- No response: host down, or ICMP blocked by firewall
- "Destination host unreachable": no route to host
- "Name resolution failure": DNS problem
```

### traceroute / tracepath — Route Discovery

```bash
traceroute google.com
tracepath google.com   # no root required

# Each line is a hop (router). Columns: hop#, hostname, RTT
 1  192.168.1.1 (192.168.1.1)   0.4 ms   # your router
 2  10.10.0.1 (10.10.0.1)       5.2 ms   # ISP first hop
 3  * * *                                # firewall blocks ICMP
 4  72.14.215.165                8.1 ms   # Google backbone
 5  142.250.80.46                12.3 ms  # destination

* * *  means that hop doesn't respond to traceroute probes
```

### dig / nslookup — DNS Diagnostics

```bash
# Basic lookup
dig example.com
dig example.com A      # IPv4
dig example.com AAAA   # IPv6
dig example.com MX     # mail servers

# Short answer only
dig +short example.com

# Query specific DNS server
dig @8.8.8.8 example.com        # Google DNS
dig @1.1.1.1 example.com        # Cloudflare DNS
dig @192.168.1.1 example.com    # Your local resolver

# Trace full resolution path
dig +trace example.com

# Reverse lookup
dig -x 93.184.216.34

# Check TTL (how long until cached record expires)
dig +ttlunits example.com
```

### curl — HTTP Diagnostics

```bash
# Basic request with headers
curl -v https://example.com

# Timing breakdown
curl -w @- -o /dev/null -s https://example.com <<'EOF'
    time_namelookup:  %{time_namelookup}s
       time_connect:  %{time_connect}s
    time_appconnect:  %{time_appconnect}s
   time_pretransfer:  %{time_pretransfer}s
      time_redirect:  %{time_redirect}s
 time_starttransfer:  %{time_starttransfer}s
                    ----------
         time_total:  %{time_total}s
EOF

# What each means:
# time_namelookup:  DNS resolution
# time_connect:     TCP handshake complete
# time_appconnect:  TLS handshake complete
# time_starttransfer: first byte received (TTFB)

# Follow redirects
curl -L https://example.com

# Set headers
curl -H "Authorization: Bearer token" https://api.example.com/users

# POST JSON
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"test"}' https://api.example.com/users

# Ignore TLS errors (debugging only, never in production)
curl -k https://self-signed.example.com

# Use specific DNS
curl --resolve example.com:443:93.184.216.34 https://example.com
```

### ss / netstat — Socket Inspection

```bash
# List all listening ports
ss -tulnp
#  -t TCP  -u UDP  -l listening  -n no name resolve  -p show process

# All established TCP connections
ss -tnp

# Connection count by state
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn

# Find what's using port 3000
ss -tulnp | grep :3000
# or:
lsof -i :3000

# Check connection to a specific remote host
ss -tn dst 8.8.8.8
```

### tcpdump — Packet Capture

```bash
# Capture all traffic on interface eth0
tcpdump -i eth0

# Capture HTTP traffic (port 80)
tcpdump -i any port 80

# Capture traffic to/from specific host
tcpdump -i any host 93.184.216.34

# Save capture to file (open in Wireshark)
tcpdump -i any -w capture.pcap

# Read from file
tcpdump -r capture.pcap

# Verbose output (see packet contents in ASCII)
tcpdump -i any -A port 8080

# Capture DNS queries
tcpdump -i any port 53

# Capture SYN packets (new connections)
tcpdump -i any 'tcp[tcpflags] & tcp-syn != 0'
```

### nc (netcat) — Raw Connectivity Testing

```bash
# Test if port is open
nc -zv example.com 443
nc -zv -w 3 example.com 443  # 3 second timeout

# Simple TCP server (for testing)
nc -l 8080

# Send HTTP request manually
echo -e 'GET / HTTP/1.0\r\nHost: example.com\r\n\r\n' | nc example.com 80

# UDP test
nc -zu example.com 53
```

## Common Scenarios

### "Connection refused"

```bash
# Port is reachable but nothing is listening
nc -zv localhost 3000
# Connection to localhost 3000 port [tcp/*] failed: Connection refused

# Debug:
ss -tulnp | grep :3000   # Is anything listening?
ps aux | grep node        # Is the process running?
journalctl -u myapp -f   # Check logs
```

### "Connection timed out"

```bash
# Packet is being dropped (firewall) — no RST received
# Takes time to fail (kernel gives up after ~2 minutes by default)

# Debug:
ping target.host            # Is host reachable at all?
traceroute target.host      # Where does it stop?
# Check firewall rules on target:
iptables -L -n | grep <port>
ufw status
```

### "Could not resolve hostname"

```bash
# DNS failure
dig google.com               # Can you resolve at all?
dig @8.8.8.8 google.com      # Works with external DNS?
cat /etc/resolv.conf          # What resolver is configured?
resolvectl status             # systemd-resolved status
```

### Docker networking issues

```bash
# Container can't reach the internet
docker exec mycontainer ping 8.8.8.8
docker exec mycontainer ping google.com

# Inspect container's network
docker inspect mycontainer | jq '.[0].NetworkSettings'

# Check if bridge network is working
ip addr show docker0
iptables -t nat -L DOCKER

# Container not reachable from host on published port
docker ps                          # Is port mapped?
ss -tulnp | grep docker-proxy      # Is docker-proxy listening?
curl http://localhost:8080          # Test from host
```

## Latency Debugging

```bash
# Identify slow phase with curl timing
curl -w "dns=%{time_namelookup} connect=%{time_connect} \
tls=%{time_appconnect} ttfb=%{time_starttransfer} \
total=%{time_total}\n" -o /dev/null -s https://api.example.com

# High dns time → DNS resolver issue, consider local caching
# High connect time → network latency, far server, or TCP issue
# High tls time → certificate chain issue, OCSP, slow crypto
# High ttfb → slow server-side processing
```

## Quick Reference Cheatsheet

```bash
# Connectivity
ping host                  # ICMP reachability
traceroute host            # Route to host
nc -zv host port           # TCP port test

# DNS
dig host                   # DNS lookup
dig +trace host            # Full resolution path
dig @8.8.8.8 host          # Query specific resolver

# Sockets
ss -tulnp                  # Listening ports + process
ss -tnp                    # Active connections
lsof -i :port              # What owns this port?

# HTTP
curl -v https://host       # Verbose HTTP request
curl -w timing...          # Timing breakdown

# Packets
tcpdump -i any port X      # Capture traffic on port
tcpdump -i any host X      # Capture traffic to/from host
```

## Practice

1. Run the curl timing command against 3 different sites. Compare DNS, connect, and TLS times.
2. Use `tcpdump -i any port 53` in one terminal and `dig google.com` in another. Watch the DNS query/response packets.
3. Run `ss -tnp` on your machine and identify which processes have open connections and to where.
