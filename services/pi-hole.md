# Pi-hole: Centralized DNS Sinkhole & DHCP Engine

> [!NOTE]
> **Legacy Infrastructure Notice:** Pi-hole served as the primary DNS sinkhole and DHCP provider during the initial iterations of this homelab environment. In the current network topology, Pi-hole has been superseded by **OPNsense (running Unbound DNS and native DHCP/SLAAC services)** to centralize perimeter routing and DNS filtering into a single firewall appliance.

## Overview & Purpose

Pi-hole serves as the primary network-wide DNS sinkhole and internal DHCP engine for the home lab environment. By acting as the authoritative DNS gateway for all local clients, it performs real-time egress telemetry suppression, blocking ad networks, tracking endpoints, and known malicious domains before outbound TCP/UDP connections can be established.

### Core Engine Architecture: FTLDNS
Pi-hole operates on **FTLDNS (Faster Than Light DNS)**, an optimized fork of `dnsmasq`. Rather than relying on traditional disk-bound logging, FTLDNS processes and stores DNS queries in shared memory (`/dev/shm`), backed by an SQLite database for persistent long-term analytics. This architecture allows:
* **In-Memory Query Matching:** Ultra-low latency regex pattern matching against multi-million-entry blocklists.
* **Low Overhead Caching:** Reduced disk I/O wear on flash storage while maintaining a minimal memory footprint.
* **Native EDNS & DNSSEC Support:** Hardware-efficient validation of upstream cryptographic DNS records.

---

## Deployment Architecture

### Docker Container Configuration

```yaml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    environment:
      TZ: Asia/Karachi
    volumes:
      - /home/$hostname/docker/pihole:/etc/pihole
    cap_add:
      - NET_ADMIN
      - SYS_NICE
    restart: unless-stopped
    shm_size: 128m
```

> **Deployment Rationale (Capabilities):**
> * **`NET_ADMIN`:** Grants permission to manipulate host network interfaces, ARP tables, and bind raw sockets required for DHCP address distribution.
> * **`SYS_NICE`:** Allows FTLDNS to adjust process scheduling priorities to prevent query dropouts under high host CPU utilization.

> [!NOTE]
> **Network Mode & Scope Note:** 
> Pi-hole was configured for IPv4 DHCP distribution only. IPv6 network auto-configuration (DHCPv6 / SLAAC / Router Advertisements) remained entirely delegated to the default edge router.

---

### Upstream DNS Strategy & Security Protocol Stack

Pi-hole routes un-cached upstream DNS queries using a dual-stack (IPv4/IPv6) multi-provider architecture:

1. **Cloudflare Upstream:**
* **IPv4:** `1.1.1.1` / `1.0.0.1`
* **IPv6:** `2606:4700:4700::1111` / `2606:4700:4700::1001`
* **Features:** Built-in DNSSEC validation.


2. **Google Upstream:**
* **IPv4:** `8.8.8.8` / `8.8.4.4`
* **IPv6:** `2001:4860:4860::8888` / `2001:4860:4860::8844`
* **Features:** Built-in DNSSEC validation and EDNS Client Subnet (ECS) support.

---

> [!IMPORTANT]
> **Upstream Resolver Selection Behavior (First-Responder Race):**
> 
> Declaring multiple upstream DNS servers in Pi-hole does **not** perform round-robin load balancing or maintain historical RTT latency profiling. FTLDNS (derived from `dnsmasq`) uses a **First-Responder Race**:
> 
> 1. **Parallel Probing:** Upon initialization or re-probe triggers, FTLDNS forwards a single query to **all** configured IPv4 and IPv6 upstream resolvers simultaneously.
> 2. **Fastest-Wins Lock:** Whichever server returns a valid response first is set as the active resolver.
> 3. **Traffic Consolidation:** **100% of subsequent queries** are routed exclusively to that winning server.
> 4. **Re-check Interval:** FTLDNS re-queries all upstream servers simultaneously to evaluate the fastest responder only after **1,000 queries** have passed, or immediately if the currently active server fails to respond.
> 
> Consequently, traffic does not balance evenly; one server will handle the vast majority of requests until a re-probe occurs or a query times out.

---

#### Cryptographic & Operational Mechanisms

* **DNSSEC (Domain Name System Security Extensions):** Prevents DNS cache poisoning and Man-in-the-Middle (MitM) spoofing attacks. Pi-hole cryptographically verifies the digital signatures ($RRSIG$) provided by upstream root zones against trusted public keys ($DNSKEY$). Unsigned or tampered responses return a `SERVFAIL` status, isolating clients from hijacked destinations.
* **ECS (EDNS Client Subnet):** An extension of the DNS protocol (RFC 7871) that forwards a truncated snippet of the client's IP address subnet (/24 for IPv4, /56 for IPv6) to the upstream resolver. This enables authoritative CDN resolvers (e.g., Akamai, Cloudflare) to return geographically optimal edge server IPs without compromising full client IP privacy.

---

### Managed Blocklists

To maintain high precision without breaking legitimate web applications, the sinkhole aggregates the following feeds:

| Feed Name | Focus / Threat Domain | Format |
| --- | --- | --- |
| **StevenBlack Hosts** | Unified baseline ad/malware feed | `hosts` file syntax |
| **HaGezi Pro** | Aggressive ad, tracking, and telemetry suppression | Domain list |
| **HaGezi Popup Ads** | Malvertising & intrusive popups | Domain list |
| **HaGezi Threat Intelligence (TIF)** | High-confidence phishing, malware, C2, and scam domains | Domain list |
| **OISD NSFW** | Explicit content domain sinkhole | Domain list |

---

## Security Rationale & Trade-offs

### Sinkhole Response Mode: `NULL` Block (RFC 1122 / RFC 4291)

Pi-hole is configured to use the **`NULL` blocking mode**, returning `0.0.0.0` (for IPv4) or `::` (for IPv6) when a blocked domain is queried.

* **Technical Advantage over `NXDOMAIN`:** Returning `NXDOMAIN` (Non-Existent Domain) can cause client operating systems to retry upstream queries repeatedly across secondary adapters.
* **Technical Advantage over IP Redirection:** Returning an internal webserver IP (to display a block page) forces the client browser to initiate a TCP connection, leading to TLS certificate mismatch errors (`NET::ERR_CERT_COMMON_NAME_INVALID`) on HTTPS connections.
* **RFC Compliance:** The `0.0.0.0` address is defined in RFC 1122 as the "unspecified" target within the local host network, while `::` is defined in RFC 4291 (Section 2.5.2) as the IPv6 Unspecified Address. Clients immediately abort the connection attempt without timeout delays or retry amplification.

### Security Trade-offs

* **Central Point of Failure:** If the container or host crashes, all downstream network resolution fails unless secondary internal resolvers are defined.
* **False Positives:** Aggressive feeds (e.g., HaGezi Pro) can break affiliate marketing links, single sign-on (SSO) redirects, or app-level telemetry required for functional rendering. Whitelisting rules must be maintained continuously.

---

## Post-Mortem: Complete IPv6 DNS Outage via Default Router WAN/RA Settings

### Incident Summary

During the initial Pi-hole deployment, dual-stack network clients suddenly lost all domain name resolution. Hostname resolution failed completely across client devices (resulting in connection timeouts), while direct IP connectivity (e.g., `ping 1.1.1.1` or browsing to a direct IP address) remained fully functional. Network diagnostics revealed that clients were attempting to route IPv6 DNS queries to the router's default interface/WAN configuration, where no local filtering DNS listener was active.

### Root Cause Analysis (RCA)

While IPv4 DHCP allocation was successfully managed by Pi-hole, dual-stack IPv6 network autoconfiguration remained handled by the default router using **DHCPv6 and Router Advertisements (RA)** (RFC 4861). 

The router's default IPv6 DNS settings were left on **"Auto / WAN Configuration"**. As a result, the router broadcasted its own Link-Local address (`fe80::1`) as the authoritative RDNSS (Recursive DNS Server) to all IPv6 clients. Because dual-stack endpoints prioritized IPv6 DNS resolution over IPv4, queries were sent to an endpoint with no functional DNS filtering daemon running, causing complete DNS resolution failure across the network.

```mermaid
graph LR
    classDef client fill:#1f4e78,stroke:#333,color:#fff;
    classDef router fill:#595959,stroke:#333,color:#fff;
    classDef error fill:#b30000,stroke:#333,color:#fff,stroke-width:2px;

    Client["📱 Client Device"]:::client
    Router["🛡️ Router Default WAN Config<br/>(fe80::1)"]:::router
    Outage["❌ Connection Timeout<br/>(DNS Resolution Failure)"]:::error

    Client -->|IPv6 DHCPv6 / RA| Router
    Router -->|No Active DNS Listener| Outage
```

### Diagnostic Process

1. **Layer 3 Connectivity Check:** Executed `ping 1.1.1.1` (IPv4) and `ping 2606:4700:4700::1111` (IPv6). Direct IP routing was functional, proving gateway interfaces and physical layers were up.
2. **DHCPv4 Scoping Verification:** Executed `ipconfig /renew` (Windows) / `dhclient -v` (Linux). Confirmed that Pi-hole was correctly issuing IPv4 parameters and setting itself as the IPv4 DNS target.
3. **IPv6 DNS Query Path Tracing:** Executed `nslookup target-domain.com`. Identified that queries failed because client network stacks defaulted to the router's unconfigured IPv6 DNS target:

```bash
# Diagnostic command revealing non-responsive IPv6 DNS server:
nslookup doubleclick.net
# Output showed:
Server:  UnKnown
Address: fe80::1%12    # <-- Router Link-Local (No Active DNS Response)

```

### Remediation & Resolution

1. **Router IPv6 DNS Override:** Navigated to the default router's IPv6 LAN / Router Advertisement settings.
2. **WAN Config to Static Pi-hole Pointer:** Changed the IPv6 DNS option from *WAN Configuration / Auto* to **Static**, explicitly inputting the **Pi-hole server's IPv6 Link-Local address (`fe80::...`)**.
3. **Validation:** Flushed client DNS caches (`ipconfig /flushdns` / `resolvectl flush-caches`) and renewed interface leases. Verified via `nslookup` that IPv6 queries were now successfully arriving at Pi-hole's IPv6 interface, resolving hostnames correctly and restoring dual-stack network functionality.
