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
| Monitoring             | Prometheus and Grafana                                      |
| Metrics exporter       | Node Exporter                                               |
| Monitoring access      | Local network only                                          |

## Security Configuration

-   SSH key-based authentication is enabled.
-   Password-based SSH authentication is disabled.
-   UFW is enabled with inbound traffic denied by default.
-   SSH is restricted to the home IPv4 subnet `192.168.1.0/24`.
-   Caddy is configured to provide the public HTTPS endpoint for Jellyfin.
-   Cloudflare is configured for DNS resolution only (grey cloud), so the home public IP address is visible and Cloudflare's HTTP proxy protections do not apply.
-   Jellyfin's port `8096` must not be forwarded directly through the router; public web traffic should reach Caddy on TCP ports `80` and `443`.
-   UFW IPv6 support is enabled, but the router's IPv6 firewall behaviour still needs to be verified.
-   Monitoring dashboards are intended to be accessible only from the trusted home network (`192.168.1.0/24`).
-   Grafana uses TCP port `3000` and Prometheus uses TCP port `9090`; these ports are only exposed to the local network.

These controls address different risks: SSH keys authenticate clients, firewall rules restrict which sources can reach services, and Caddy provides HTTPS for the public Jellyfin endpoint.

Caddy successfully obtained a Let's Encrypt certificate for `jellyfin.sfmcmullen.com` on 13 October 2026. External access and playback should still be tested from a device outside the home network.

## Network and Services

The Windows administration PC connects to the Ubuntu server over the home network. The server runs Docker, which hosts Jellyfin in a container. Caddy runs directly on the Ubuntu host and reverse-proxies requests for `jellyfin.sfmcmullen.com` to `192.168.1.222:8096`.

The Docker-published Jellyfin endpoint is bound to the server's Wi-Fi address rather than all host interfaces. Docker also manages its own networking and firewall rules, which must be considered when assessing service exposure.

See [`docs/network-topology.md`](docs/network-topology.md) for addressing, connectivity tests, and configuration details. See [`docs/public-jellyfin.md`](docs/public-jellyfin.md) for the Cloudflare DNS, Caddy, HTTPS, router forwarding, verification, and recovery details.

## Monitoring and Observability

Prometheus and Grafana have been deployed on the Ubuntu server to provide infrastructure monitoring and visualisation. Prometheus collects time-series metrics from configured monitoring targets, while Grafana provides dashboards for inspecting the collected data. The monitoring setup provides a foundation for investigating server performance, identifying resource constraints, and understanding the operational state of the home lab.

## Project Objectives

-   Administer a headless Linux server remotely.
-   Understand IPv4 addressing, subnetting, gateways, DNS, and network interfaces.
-   Configure SSH authentication and host firewall rules.
-   Deploy and manage services using Docker.
-   Understand Docker port publishing and service reachability.
-   Document infrastructure and troubleshooting decisions.
-   Deploy Prometheus and Grafana for infrastructure monitoring.
-   Configure and interpret monitoring metrics and dashboards.
-   Develop Python tools for network and service health checks.
-   Experiment with routing protocols and network automation in a virtual lab.

## Future Development

1. Configure monitoring alerts for resource usage and service failures.
2. Build a Python health-check tool for DNS, network connectivity, and Jellyfin availability.
3. Record response times and failures in timestamped logs.
4. Add automated tests for the monitoring logic.
5. Test public Jellyfin access and media playback from outside the home network.
6. Verify the router's IPv6 firewall policy and review public service exposure.
7. Investigate dynamic DNS updates if the home public IPv4 address changes.
8. Build a virtual networking lab to practise static routing, OSPF, and BGP.

## Skills Demonstrated

-   Linux system administration
-   TCP/IP networking and subnetting
-   SSH authentication and firewall configuration
-   Docker container deployment and management
-   DNS configuration and reverse proxy setup
-   HTTPS certificate management
-   Infrastructure monitoring with Prometheus and Grafana
-   Monitoring dashboard configuration and metric interpretation
-   Network troubleshooting and technical documentation

## Project Status

**Core infrastructure configured, with monitoring implemented.**

The Ubuntu server is managed remotely over SSH using key-based authentication. UFW is enabled, and Jellyfin runs in Docker with
a read-only media mount for the external SSD.

Caddy provides HTTPS access to Jellyfin through the configured public hostname, with certificates managed automatically by Let's Encrypt.

Prometheus and Grafana have also been deployed, and the monitoring dashboard is operational.

Remaining work includes verifying that monitoring interfaces are restricted to the local network, testing Jellyfin access and playback externally, reviewing the router's IPv6 firewall behaviour, and developing additional monitoring automation.

## Lessons Learned

This project has provided practical experience in configuring and managing a Linux server, deploying containerised applications, and making a self-hosted service accessible through DNS, port forwarding, a reverse proxy, and HTTPS.

It has also highlighted the importance of understanding service bindings, firewall rules, and Docker's networking behaviour when controlling access to internal services.

Prometheus and Grafana extend the project by providing a way to observe infrastructure metrics and investigate performance issues. Further work will focus on alerting, automated health checks, troubleshooting documentation, and testing recovery procedures.
