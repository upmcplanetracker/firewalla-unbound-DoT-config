Unbound DoT Custom Configuration for Firewalla
==============================================

This supplemental configuration enables DNS over TLS (DoT) for your Firewalla device, encrypting DNS queries between your Firewalla and upstream resolvers for improved privacy.

> **Note**: DNS traffic between the upstream resolver and authoritative root servers remains unencrypted—this is standard for virtually all DNS resolutions, regardless of setup.

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

Installation
------------

1.  **SSH** into your Firewalla device
2.  **Create** the configuration directory (if it doesn't exist):
    
        mkdir -p ~/.firewalla/config/unbound_local/
    
3.  **Create** `unbound_custom.conf` in that directory:
    
        nano ~/.firewalla/config/unbound_local/unbound_custom.conf
    
4.  **Paste** the configuration contents into the file and save
5.  **Restart** Unbound:
    
        sudo systemctl restart unbound
    
    Or toggle DNS off/on in the Firewalla app
    

Verification
------------

Check that Unbound is running:

    sudo systemctl status unbound

To see detailed query activity (temporary):

1.  Uncomment `verbosity: 4` in the config
2.  Restart Unbound
3.  Browse websites and check logs: `sudo journalctl -u unbound -f`
4.  Reset verbosity to `1` (or comment it out) when done

Customization
-------------

### IPv6 Configuration

**IPv6 is disabled by default** in this configuration. If your network supports IPv6:

1.  Uncomment `do-ip6: yes`
2.  Also uncomment the IPv6 `forward-addr` entries for your chosen providers
3.  Optionally, uncomment `prefer-ip6: yes` to prioritize IPv6 over IPv4
4.  Restart Unbound

**If you don't have IPv6**, leave `# do-ip6: yes` commented out—this prevents unnecessary connection attempts and avoids potential resolution delays.

### Adding/Removing DNS Servers

Edit the `forward-addr` lines to use your preferred DoT providers. Format: `IP_ADDRESS@PORT#SERVER_HOSTNAME`

Additional providers:

*   **Control-D**: `76.76.2.0@853#dns.controld.com` / `76.76.10.0@853#dns.controld.com`
*   **NextDNS**: `45.90.28.0@853#dns.nextdns.io` / `45.90.30.0@853#dns.nextdns.io`

More servers: [DNS Privacy Project](https://dnsprivacy.org/wiki/display/DP/Public+Resolvers)

Memory & Cache Tuning
---------------------

The configuration includes commented cache settings:

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
| Gold/Orange | `64m`/`128m` max |
| Purple, Purple SE, Red, Blue | **Keep commented out** (limited RAM) |

> **⚠️ Important**: Always monitor memory usage with `htop` after changes while generating network traffic.

### Hardware Recommendations

Device

Recommended Limit

Gold Pro

Up to `256m`/`512m`

Gold/Orange

`64m`/`128m` max

Purple, Purple SE, Red, Blue

**Keep commented out** (limited RAM)

**⚠️ Important**: Always monitor memory usage with `htop` after changes while generating network traffic.

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
    

Configuration File
------------------

    server:
        tls-cert-bundle: "/etc/ssl/certs/ca-certificates.crt"
        # msg-cache-size: 4m
        # rrset-cache-size: 8m
        prefetch: yes
        prefetch-key: yes
        # do-ip6: yes
        # verbosity: 1
        # prefer-ip4: no
        # prefer-ip6: no
    
    forward-zone:
        name: "."
        forward-first: yes
        forward-tls-upstream: yes
    
        # Cloudflare (IPv4)
        forward-addr: 1.1.1.1@853#cloudflare-dns.com
        forward-addr: 1.0.0.1@853#cloudflare-dns.com
        # Cloudflare (IPv6 - uncomment if you have IPv6)
        # forward-addr: 2606:4700:4700::1111@853#cloudflare-dns.com
        # forward-addr: 2606:4700:4700::1001@853#cloudflare-dns.com
    
        # Google (IPv4)
        forward-addr: 8.8.8.8@853#dns.google
        forward-addr: 8.8.4.4@853#dns.google
        # Google (IPv6 - uncomment if you have IPv6)
        # forward-addr: 2001:4860:4860::8888@853#dns.google
        # forward-addr: 2001:4860:4860::8844@853#dns.google
    
        # Quad9 (IPv4)
        forward-addr: 9.9.9.9@853#dns.quad9.net
        forward-addr: 149.112.112.112@853#dns.quad9.net
        # Quad9 (IPv6 - uncomment if you have IPv6)
        # forward-addr: 2620:fe::fe@853#dns.quad9.net
        # forward-addr: 2620:fe::9@853#dns.quad9.net
    

* * *

Disclaimer
----------

Not affiliated with or endorsed by Firewalla. Use at your own risk—testing on a non-production device first is recommended.
