# Home Network Lab

A self-hosted infrastructure project using Ubuntu Server, Docker, and network security controls. The project is designed to develop practical skills in Linux administration, TCP/IP networking, service deployment, troubleshooting, and network automation.

## Project Overview

I repurposed an old PC into a headless Ubuntu Server, managed remotely from my Windows PC over SSH. The server runs Jellyfin in Docker, providing a self-hosted media service. Caddy runs on the Ubuntu host as a reverse proxy for HTTPS access through `jellyfin.sfmcmullen.com`.

I configured SSH key-based authentication and disabled password-based SSH login. UFW is enabled to restrict inbound traffic, including SSH access to the trusted home IPv4 subnet.

The project is being developed incrementally, with an emphasis on understanding the underlying network configuration, documenting decisions, and building tools to monitor and automate infrastructure.

## Infrastructure

| Component              | Configuration                                               |
| ---------------------- | ----------------------------------------------------------- |
| Server hostname        | `mcmullen-lab`                                              |
| Operating system       | Ubuntu Server 26.04.1 LTS                                   |
| Administration client  | Windows PC                                                  |
| Remote administration  | SSH with key-based authentication                           |
| Network connectivity   | Wi-Fi via TP-Link Archer T9UH                               |
| Server Wi-Fi address   | `192.168.1.222`                                             |
| Home router            | `192.168.1.254`                                             |
| Firewall               | UFW                                                         |
| Container platform     | Docker                                                      |
| Hosted service         | Jellyfin                                                    |
| Jellyfin host endpoint | `192.168.1.222:8096`                                        |
| Reverse proxy          | Caddy, running as a systemd service                         |
| Public hostname        | `jellyfin.sfmcmullen.com`                                   |
| DNS                    | Cloudflare DNS-only A record                                |
| HTTPS certificate      | Let's Encrypt, managed automatically by Caddy               |
| Media storage          | External 2 TB exFAT drive mounted read-only at `/srv/media` |

## Security Configuration

-   SSH key-based authentication is enabled.
-   Password-based SSH authentication is disabled.
-   UFW is enabled with inbound traffic denied by default.
-   SSH is restricted to the home IPv4 subnet `192.168.1.0/24`.
-   Caddy is configured to provide the public HTTPS endpoint for Jellyfin.
-   Cloudflare is configured for DNS resolution only (grey cloud), so the home public IP address is visible and Cloudflare's HTTP proxy protections do not apply.
-   Jellyfin's port `8096` must not be forwarded directly through the router; public web traffic should reach Caddy on TCP ports `80` and `443`.
-   UFW IPv6 support is enabled, but the router's IPv6 firewall behaviour still needs to be verified.

These controls address different risks: SSH keys authenticate clients, firewall rules restrict which sources can reach services, and Caddy provides HTTPS for the public Jellyfin endpoint.

Caddy successfully obtained a Let's Encrypt certificate for `jellyfin.sfmcmullen.com` on 13 October 2026. External access and playback should still be tested from a device outside the home network.

## Network and Services

The Windows administration PC connects to the Ubuntu server over the home network. The server runs Docker, which hosts Jellyfin in a container. Caddy runs directly on the Ubuntu host and reverse-proxies requests for `jellyfin.sfmcmullen.com` to `192.168.1.222:8096`.

The Docker-published Jellyfin endpoint is bound to the server's Wi-Fi address rather than all host interfaces. Docker also manages its own networking and firewall rules, which must be considered when assessing service exposure.

See [`docs/network-topology.md`](docs/network-topology.md) for addressing, connectivity tests, and configuration details. See [`docs/public-jellyfin.md`](docs/public-jellyfin.md) for the Cloudflare DNS, Caddy, HTTPS, router forwarding, verification, and recovery details.

## Project Objectives

-   Administer a headless Linux server remotely.
-   Understand IPv4 addressing, subnetting, gateways, DNS, and network interfaces.
-   Configure SSH authentication and host firewall rules.
-   Deploy and manage services using Docker.
-   Understand Docker port publishing and service reachability.
-   Document infrastructure and troubleshooting decisions.
-   Develop Python tools for network and service monitoring.
-   Experiment with routing protocols and network automation in a virtual lab.

## Future Development

1. Build a Python monitoring tool for network connectivity, DNS, and Jellyfin availability.
2. Record response times and failures in timestamped logs.
3. Add automated tests for the monitoring logic.
4. Build a virtual networking lab to practise static routing, OSPF, and BGP.
5. Test public Jellyfin access and media playback from outside the home network.
6. Verify the router's IPv6 firewall policy and review public service exposure.
7. Investigate dynamic DNS updates if the home public IPv4 address changes.

## Skills Demonstrated

-   Linux system administration
-   TCP/IP networking and subnetting
-   SSH and firewall configuration
-   Docker service deployment
-   DNS configuration and reverse proxy setup
-   HTTPS certificate management
-   Network troubleshooting
-   Technical documentation
-   Planned Python automation and monitoring

## Project Status

**Core infrastructure configured.** The Ubuntu server is accessible over SSH using key-based authentication, UFW is enabled, and Jellyfin runs in Docker with a read-only media mount. Caddy is configured as a reverse proxy for `jellyfin.sfmcmullen.com`, and its logs confirm that a Let's Encrypt HTTPS certificate was obtained successfully.

Remaining checks include testing the Jellyfin page and media playback from outside the home network and verifying the router's IPv6 firewall behaviour.

Further development will focus on network monitoring, automation, and virtual networking experiments.

## Lessons Learned

This section will be expanded with practical examples of troubleshooting, configuration decisions, testing, and the reasoning behind security controls.
