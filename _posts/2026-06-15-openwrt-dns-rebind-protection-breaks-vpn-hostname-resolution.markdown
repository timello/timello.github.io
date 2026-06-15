---
layout: post
title:  "When OpenWrt DNS Rebind Protection Breaks VPN Hostname Resolution"
date:   2026-06-15 17:30:00 +0200
categories: networking dns vpn openwrt
---
I recently ran into a confusing DNS issue while using a VPN client on macOS. Applications behind the VPN worked correctly when I configured my computer to use a public DNS server, but failed when I used my OpenWrt router as the DNS resolver.

At first glance, it looked like a VPN or DNS forwarding problem. However, the root cause turned out to be OpenWrt's DNS rebinding protection.

## The Symptoms

* VPN-connected applications could not resolve certain hostnames.
* Using the router as the DNS server resulted in `NXDOMAIN` responses.
* Using a public DNS server worked as expected.
* The issue only affected specific hostnames.

For example:

```bash
nslookup internal-service.example.com 192.168.1.1
```

would fail, while:

```bash
nslookup internal-service.example.com 8.8.8.8
```

would successfully return IP addresses.

## Initial Investigation

The first assumption was that the VPN client was providing custom DNS servers or split-DNS configuration. However, inspecting the system's DNS configuration showed that only the router was being used as a resolver.

Next, I tested the hostname directly from the router itself. Surprisingly, the router was able to resolve the hostname correctly.

This suggested that the issue was not upstream DNS resolution, but rather how the router handled responses before passing them to clients.

## The Clue

Checking the OpenWrt logs revealed repeated messages like:

```text
possible DNS-rebind attack detected: internal-service.example.com
```

This immediately pointed to DNS rebinding protection.

## What Is DNS Rebinding Protection?

DNS rebinding protection is a security feature in `dnsmasq` that helps prevent malicious websites from resolving public domain names to private network addresses.

For example, if a public hostname resolves to:

```text
10.x.x.x
172.16.x.x - 172.31.x.x
192.168.x.x
```

`dnsmasq` may consider the response suspicious and block it.

In many VPN environments, however, this behavior is entirely legitimate. Public hostnames are often used to point to private resources accessible only through the VPN.

## Confirming the Cause

On OpenWrt, the setting can be checked with:

```bash
uci get dhcp.@dnsmasq[0].rebind_protection
```

If enabled, the logs can be inspected with:

```bash
logread | grep -i rebind
```

In my case, the logs clearly showed that the DNS responses were being blocked.

## The Fix

Rather than disabling DNS rebinding protection globally, a safer approach is to whitelist trusted domains.

For example:

```bash
uci add_list dhcp.@dnsmasq[0].rebind_domain='example.com'
uci commit dhcp
/etc/init.d/dnsmasq restart
```

This allows specific domains to resolve to private IP addresses while keeping rebinding protection enabled for everything else.

## Lessons Learned

When troubleshooting VPN-related DNS issues, don't assume the VPN is at fault. If you're using OpenWrt, Adblock, or `dnsmasq`, check whether DNS rebinding protection is silently blocking responses.

A few log messages saved hours of chasing the wrong problem.

Sometimes the DNS server is doing exactly what it was designed to do—it's just protecting you from a threat that happens to look very similar to a legitimate VPN configuration.
</content>
</invoke>
