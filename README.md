# Unbound DoT Custom Configuration for Firewalla

This is a supplemental `unbound` configuration for your Firewalla device. It enhances your network's privacy and security by enabling DNS over TLS (DoT), with a robust fallback to standard DNS resolution if DoT servers are unavailable.

This configuration forwards all DNS queries to a list of trusted DoT resolvers.

---

## How It Works 🧐

* **`forward-tls-upstream: yes`**: This is the key directive that tells `unbound` to encrypt DNS queries using TLS.
* **`forward-addr`**: This specifies the DoT servers to use. The format is `IP_ADDRESS@PORT#SERVER_HOSTNAME`. The hostname is used to validate the TLS certificate, ensuring you're talking to the correct server.
* **Server Selection**: You can list multiple DoT providers (like Cloudflare, Google, and Quad9 in the example). `unbound` is smart—it monitors the performance of these servers and will prioritize using the fastest, most responsive one for your queries.
* **Fallback**: The `forward-first: yes` setting tells `unbound` to try the DoT forwarders first. If for some reason all of them are unreachable, `unbound` will automatically fall back to its default behavior of resolving domains itself (acting as a recursive resolver). This ensures your internet connection never breaks due to a DoT outage.

---

## Installation 🛠️

1.  SSH into your Firewalla device.
2.  Create the file `unbound_custom.conf` in the following directory: `~/.firewalla/config/unbound_local/`.
3.  Copy the contents of this configuration into the new file.
4.  Restart your Firewalla's DNS service from the app, or simply reboot the device for the changes to take effect.

---

## Customization ⚙️

You are free to customize the list of DNS servers. You can add, remove, or change any of the `forward-addr` lines to use your preferred DoT providers. Just ensure you follow the correct format.

A good resource for finding more DoT servers is the [DNS Privacy Project](https://dnsprivacy.org/public_resolvers/).

---

## ⚠️ A Critical Note on Memory and Cache Configuration

This configuration includes two lines to increase the DNS cache size:
server:
`msg-cache-size: 256m`
`rrset-cache-size: 512m`

These settings can significantly speed up browsing by storing more DNS responses in memory. However, allocating too much memory can crash the DNS service or make your Firewalla unstable.

### Tuning a Firewalla Gold / Gold Plus
If you have a device with ample memory, follow these steps carefully:

1.  **Start with the cache disabled.** Comment out these two lines with a `#` to begin:
    ```
    # msg-cache-size: 256m
    # rrset-cache-size: 512m
    ```
2.  **Monitor your system.** Connect via SSH and run `htop` or `top` to check your baseline memory usage.
3.  **Enable with small values.** Uncomment the lines and start with very small values. A good starting point is `4m` and `8m`:
    ```
    msg-cache-size: 4m
    rrset-cache-size: 8m
    ```
4.  **Increase slowly.** If your system remains stable after a day, you can slowly increase the values. Try doubling them (`8m`/`16m`, then `16m`/`32m`) and monitor memory usage at each step.
5.  **Find your limit.** The `256m`/`512m` values in the example are **extremely aggressive** and should only be used on a device like the Firewalla Gold Plus with a lot of spare RAM. For most other devices, a value between `16m`/`32m` and `64m`/`128m` is more than sufficient.

### A Warning for Firewalla Purple / Purple SE Users
The **Firewalla Purple and Purple SE** have limited RAM. It is **strongly recommended** that you keep the cache settings **commented out** on these devices. Enabling a large memory cache is very likely to cause instability. If you do experiment, use only the smallest values (`4m`/`8m`) and monitor memory very closely.
