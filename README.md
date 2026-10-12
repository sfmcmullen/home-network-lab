# Home Network Lab

A self-hosted infrastructure project using Ubuntu Server, Docker, and network security controls. The project is designed to develop practical skills in Linux administration, TCP/IP networking, service deployment, troubleshooting, and network automation.

## Project Overview

I repurposed an old PC into a headless Ubuntu Server, managed remotely from my Windows PC over SSH. The server runs Jellyfin in Docker, providing a self-hosted media service on my home network.

I configured SSH key-based authentication and disabled password-based SSH login. UFW is enabled to restrict inbound traffic, including SSH access to the trusted home IPv4 subnet.

The project is being developed incrementally, with an emphasis on understanding the underlying network configuration, documenting decisions, and building tools to monitor and automate infrastructure.

## Infrastructure

| Component              | Configuration                     |
| ---------------------- | --------------------------------- |
| Server hostname        | `mcmullen-lab`                    |
| Operating system       | Ubuntu Server 26.04.1 LTS         |
| Administration client  | Windows PC                        |
| Remote administration  | SSH with key-based authentication |
| Network connectivity   | Wi-Fi via TP-Link Archer T9UH     |
| Server Wi-Fi address   | `192.168.1.222`                   |
| Home router            | `192.168.1.254`                   |
| Firewall               | UFW                               |
| Container platform     | Docker                            |
| Hosted service         | Jellyfin                          |
| Jellyfin host endpoint | `192.168.1.222:8096`              |
| Media storage          | Dedicated SSD                     |

## Security Configuration

-   SSH key-based authentication is enabled.
-   Password-based SSH authentication is disabled.
-   UFW is enabled with inbound traffic denied by default.
-   SSH is restricted to the home IPv4 subnet `192.168.1.0/24`.
-   Jellyfin is accessed over the local network; public access has not been configured.

These controls address different risks: SSH keys authenticate clients, while firewall rules restrict which sources can reach the service.

## Network and Services

The Windows administration PC connects to the Ubuntu server over the home network. The server runs Docker, which hosts Jellyfin in a container.

The Docker-published Jellyfin endpoint is bound to the server's Wi-Fi address rather than all host interfaces. Docker also manages its own networking and firewall rules, which must be considered when assessing service exposure.

See [`docs/network-topology.md`](docs/network-topology.md) for addressing, connectivity tests, and configuration details.

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
5. Investigate secure remote access and HTTPS for self-hosted services, if required.

## Skills Demonstrated

-   Linux system administration
-   TCP/IP networking and subnetting
-   SSH and firewall configuration
-   Docker service deployment
-   Network troubleshooting
-   Technical documentation
-   Planned Python automation and monitoring

## Project Status

**Initial infrastructure configured.** The Ubuntu server is accessible over SSH using key-based authentication, UFW is enabled, and Jellyfin runs in Docker for local-network access.

Further development will focus on network monitoring, automation, and virtual networking experiments.

## Lessons Learned

This section will be expanded with practical examples of troubleshooting, configuration decisions, testing, and the reasoning behind security controls.
