Unbound DoT Custom Configuration for Firewalla
==============================================

This supplemental configuration enables DNS over TLS (DoT) for your Firewalla device, encrypting DNS queries between your Firewalla and upstream resolvers for improved privacy.

> **Note**: DNS traffic between the upstream resolver and authoritative root servers remains unencrypted—this is standard for virtually all DNS resolutions, regardless of setup.

Background Info
---------------

The stock unbound conf file is located `/home/pi/.firewalla/run/unbound/unbound.conf`.  It is editable but will get continually overwritten by Firewalla.
```
server:
    # If no logfile is specified, syslog is used
    #logfile: ""
    username: ""
    chroot: ""
    directory: "/home/pi/.firewalla/run/unbound"
    pidfile: "/home/pi/.firewalla/run/unbound/unbound.pid"
    do-daemonize: no

    verbosity: 0

    interface: 127.0.0.1
    access-control: 127.0.0.1/32 allow
    port: 8953

    do-ip4: yes
    do-ip6: yes

    prefer-ip4: yes
    prefer-ip6: no

    do-udp: yes
    do-tcp: yes


    # Reduce latency
    serve-expired: yes

    msg-cache-size: 4m
    rrset-cache-size: 8m

    num-threads: 1
    outgoing-range: 700
    num-queries-per-thread: 350

    outgoing-num-tcp: 150
    incoming-num-tcp: 100

    root-hints: "/home/pi/.firewalla/run/unbound/root.hints"
    # Trust glue only if it is within the server's authority
    harden-glue: yes

    auto-trust-anchor-file: "/home/pi/.firewalla/run/unbound/root.key"
    # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
    harden-dnssec-stripped: yes

    # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
    # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
    use-caps-for-id: no

    # Reduce EDNS reassembly buffer size to avoid IP fragmentation.
    edns-buffer-size: 1232

    # Perform prefetching of close to expired message cache entries
    # This only applies to domains that have been frequently queried
    prefetch: yes
    prefetch-key: yes

    # minimum wait time for responses, increase if uplink is long. In msec.
    infra-cache-min-rtt: 500

    # Ensure privacy of local IP ranges
    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10

remote-control:
    control-enable: yes
    control-interface: 127.0.0.1
    control-port: 9053
    control-use-cert: no

include: /home/pi/.firewalla/config/unbound_local/*
```
As you can see by the include: line on the bottom, it loads all the parameters above and then tells Unbound to search in the local/user editable directory.  If the same settings are in the user config file, it will replace the current ones with those.

* * *

How It Works
------------

This configuration tells Unbound to forward all DNS queries to trusted DoT resolvers instead of performing recursive resolution itself. Here's what each part does:

*   **`forward-tls-upstream: yes`** — This is the magic line. It tells Unbound to encrypt every forwarded query using TLS.
*   **`forward-addr: IP@PORT#HOSTNAME`** — These are your upstream DoT servers. The `#HOSTNAME` part is critical—Unbound uses it to validate the server's TLS certificate, ensuring you're actually talking to Cloudflare, Google, or Quad9 and not some imposter.
*   **`forward-first: yes`** — Try the DoT forwarders first. If _all_ of them are unreachable (rare, but possible), Unbound falls back to acting as a traditional recursive resolver. Your internet keeps working.

**Smart Server Selection**: Unbound doesn't just pick the first server in the list. It measures response times and automatically routes your queries to the fastest, most reliable server at any given moment.

**DNSSEC**: Firewalla's Unbound already has DNSSEC validation enabled. You don't need to add anything to this config. Verify it's working with:

    dig sigok.verteiltesysteme.net @127.0.0.1 -p 8953
    dig sigfail.verteiltesysteme.net @127.0.0.1 -p 8953
    

The first should return `NOERROR`, the second should return `SERVFAIL`.

* * *

Installation
------------

1.  **SSH** into your Firewalla device
2.  **Create** the configuration directory (if it doesn't exist):
    
        mkdir -p ~/.firewalla/config/unbound_local/
    
3.  **Create** `unbound_custom.conf` in that directory:
    
        sudo unalias apt
        sudo apt update && sudo apt install nano
        # never run sudo apt upgrade as this will break your firewalla
        nano ~/.firewalla/config/unbound_local/unbound_custom.conf
    
5.  **Paste** the configuration contents into the file and save
6.  **Restart** Unbound:
    
        sudo systemctl restart unbound
    
    Or toggle DNS off/on in the Firewalla app

* * *

Verification
------------

Check that Unbound is running:

    sudo systemctl status unbound

To see detailed query activity (temporary):

1.  Uncomment `verbosity: 4` in the config
2.  Restart Unbound
3.  Browse websites and check logs: `sudo journalctl -u unbound -f`
4.  Reset verbosity to `1` (or comment it out) when done

* * *

Customization
-------------

### IPv6 Configuration

**IPv6 is enabled by default in the stock conf file** in this configuration. If your network supports IPv6:

1. Uncomment the IPv6 `forward-addr` entries for your chosen providers
2. Restart Unbound

**Important**: Leave `prefer-ip4: no` as-is (commented out or set to `no`). The stock conf file already has `prefer-ip6: no`.  When both are set to `no`, Unbound will use the fastest performing server regardless of protocol. If you set one to `yes`, Unbound will stubbornly prefer that protocol even when the other is performing better—which can actually hurt performance.

**If you don't have IPv6**, you don't need to change anything.

### Adding/Removing DNS Servers

Edit the `forward-addr` lines to use your preferred DoT providers. Format: `IP_ADDRESS@PORT#SERVER_HOSTNAME`

Additional providers:

*   **Control-D**: `76.76.2.0@853#dns.controld.com` / `76.76.10.0@853#dns.controld.com`
*   **NextDNS**: `45.90.28.0@853#dns.nextdns.io` / `45.90.30.0@853#dns.nextdns.io`

More servers: [DNS Privacy Project](https://dnsprivacy.org/public_resolvers/#dns-over-tls-dot)

* * *

Memory & Cache Tuning
---------------------

The configuration includes commented cache settings that are already implemented in the stock unbound conf file:

    # msg-cache-size: 4m
    # rrset-cache-size: 8m
    

**These can significantly improve browsing speed** but must be set carefully.

| Step | Action |
|------|--------|
| 1 | Monitor baseline memory: `htop` or `top` while browsing |
| 2 | Start with small values: `4m` / `8m` |
| 3 | Increase slowly: `8m`/`16m` → `16m`/`32m` |
| 4 | Stop at `64m`/`128m` for most devices |

### Hardware Recommendations

| Device | Recommended Limit |
|--------|-------------------|
| Gold Pro | Up to `256m`/`512m` |
| Gold, Plus, SE /Orange | `64m`/`128m` max |
| Purple, Purple SE, Red, Blue | **Keep commented out** (limited RAM) |

> **⚠️ Important**: Always monitor memory usage with `htop` after changes while generating network traffic.

* * *

Troubleshooting
---------------

### Unbound Won't Start

If Unbound fails after configuration changes:

    rm -R ~/.firewalla/config/unbound_local/*
    sudo systemctl restart unbound
    

This restores the default Firewalla Unbound configuration.

### Verify DoT is Working

    # Check for TLS usage (TC flag = 1 indicates truncation, common with TLS)
    dig google.com @127.0.0.1 -p 8953
    
    # Verify no DNS leaks to your ISP
    dig whoami.akamai.net @127.0.0.1 -p 8953
    
* * *

Monitoring
----------

To see the top 40 blocks over the past week: `sudo journalctl -u unbound --since "7 days ago" --no-pager | grep always_null | awk '{for(i=1;i<=NF;i++) if($i=="info:") print $(i+1)}' | sort | uniq -c | sort -nr | head -n 40`

To see the top 20 blocks over the past 24 hours: `sudo journalctl -u unbound --since "24 hours ago" --no-pager | grep always_null | awk '{for(i=1;i<=NF;i++) if($i=="info:") print $(i+1)}' | sort | uniq -c | sort -nr | head -n 20`

To watch blocks live: `sudo journalctl -u unbound -f -o cat | grep --line-buffered always_null` or `sudo journalctl -u unbound -f -o cat | grep --line-buffered -v "Empty markkey"`


* * *

Configuration File
------------------

```
server:
    tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"
    # msg-cache-size: 4m
    # rrset-cache-size: 8m
    verbosity: 1
    # stock conf default is 0
    log-local-actions: yes
    prefer-ip4: no
    # stock conf default is yes with "prefer-ip6: no" which will degrade DS performance
    harden-algo-downgrade: yes
    harden-below-nxdomain: yes
    # stock conf alreay has "harden-glue: yes" and "harden-dnssec-stripped: yes", these further increase security

forward-zone:
    name: "."
    forward-first: yes
    forward-tls-upstream: yes

    # Cloudflare
    forward-addr: 1.1.1.1@853#cloudflare-dns.com
    forward-addr: 1.0.0.1@853#cloudflare-dns.com
    # forward-addr: 2606:4700:4700::1111@853#cloudflare-dns.com
    # forward-addr: 2606:4700:4700::1001@853#cloudflare-dns.com

    # Google
    forward-addr: 8.8.8.8@853#dns.google
    forward-addr: 8.8.4.4@853#dns.google
    # forward-addr: 2001:4860:4860::8888@853#dns.google
    # forward-addr: 2001:4860:4860::8844@853#dns.google

    # Quad9
    forward-addr: 9.9.9.9@853#dns.quad9.net
    forward-addr: 149.112.112.112@853#dns.quad9.net
    # forward-addr: 2620:fe::fe@853#dns.quad9.net
    # forward-addr: 2620:fe::9@853#dns.quad9.net
```
    
* * *

Disclaimer
----------

Not affiliated with or endorsed by Firewalla. Use at your own risk—testing on a non-production device first is recommended.
