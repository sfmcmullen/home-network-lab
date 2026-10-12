# Network Topology and Configuration

## 1. Overview

The lab consists of a home LAN, a Windows administration PC, an Ubuntu Server host, and a Docker container running Jellyfin.

The server is managed remotely over SSH using key-based authentication. UFW restricts inbound access at the host level. Docker publishes Jellyfin on the server's Wi-Fi address for local-network access.

## 2. Network Topology

```text
                    Internet
                        |
                  Home Router
                  192.168.1.254
                        |
                 Home LAN: 192.168.1.0/24
                        |
              +---------+----------+
              |                    |
         Windows PC           Ubuntu Server
         192.168.1.85         mcmullen-lab
              |               192.168.1.222
              |                    |
              +---- SSH ----------+
                   TCP 22          |
             SSH key required      |
                            UFW host firewall
                                   |
                              Docker Engine
                                   |
                           Docker bridge network
                                   |
                           Jellyfin container
                             172.18.0.2
                              TCP 8096
```

The Windows PC administers the server over SSH. Devices on the home LAN can access Jellyfin through the server's published endpoint, `192.168.1.222:8096`.

The Docker container uses a Docker-managed network. Its container IP is not the same as the server's LAN address.

## 3. IP Addressing

| Device or service   | Address         | Purpose               |
| ------------------- | --------------- | --------------------- |
| Home router         | `192.168.1.254` | Default gateway       |
| Windows PC          | `192.168.1.85`  | Administration client |
| Ubuntu server Wi-Fi | `192.168.1.222` | Host LAN address      |
| Jellyfin container  | `172.18.0.2`    | Docker bridge address |

The home LAN uses the IPv4 subnet `192.168.1.0/24`, equivalent to subnet mask `255.255.255.0`.

The server's Wi-Fi interface is `wlx28ee521c9351`.

The container IP is an observed value and may change if the container or Docker network is recreated. The host address and published port are generally the better values to use when connecting from the LAN.

## 4. Remote Administration and SSH Security

SSH is used to administer the server without a directly connected monitor or keyboard.

| Setting                      | Configuration    |
| ---------------------------- | ---------------- |
| Server                       | `192.168.1.222`  |
| SSH port                     | TCP `22`         |
| Authentication               | SSH key-based    |
| Password authentication      | Disabled         |
| Permitted IPv4 source subnet | `192.168.1.0/24` |

SSH key authentication controls which clients can authenticate. UFW provides an additional restriction by limiting the IPv4 sources permitted to connect to TCP port 22.

Useful checks:

```bash
# Check the SSH service
sudo systemctl status ssh

# Check effective SSH authentication settings
sudo sshd -T | grep -E '^(pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication) '

# Check the firewall rules
sudo ufw status numbered
```

The effective SSH settings should show public-key authentication enabled and password authentication disabled. Keyboard-interactive authentication should also be checked to ensure that another password-based login path is not enabled.

The intended firewall policy permits SSH from `192.168.1.0/24`. Verify the current UFW rules before relying on this description.

## 5. Firewall Configuration

UFW is enabled with inbound traffic denied by default and outbound traffic allowed by default.

The intended policy is:

| Traffic               | Policy                           |
| --------------------- | -------------------------------- |
| SSH, TCP 22           | Allow from `192.168.1.0/24`      |
| Jellyfin, TCP 8096    | Local-network access as required |
| Other inbound traffic | Deny unless explicitly required  |

Inspect the current rules:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

Docker manages its own networking and may create firewall rules for published container ports. Therefore, UFW rules alone should not be treated as proof that a Docker-published service is inaccessible from other interfaces or networks.

No public Jellyfin access or router port forwarding is configured as part of the current setup.

## 6. Docker and Jellyfin

Jellyfin runs in a Docker container named `jellyfin`.

The observed port mapping is:

```text
Host:      192.168.1.222:8096
Container: 172.18.0.2:8096
Protocol:  TCP
```

Useful diagnostic commands:

```bash
# Running containers and published ports
sudo docker ps

# Inspect Jellyfin's published ports
sudo docker inspect -f '{{json .HostConfig.PortBindings}}' jellyfin

# Inspect container logs
sudo docker logs --tail 100 jellyfin

# Inspect host listening sockets
sudo ss -tulpn
```

The published endpoint allows Jellyfin to be accessed from the home LAN. Public accessibility is a separate matter and depends on routing, router configuration, firewall behaviour, and any remote-access setup.

## 7. Storage and Persistence

Jellyfin uses a dedicated SSD for media storage and configuration data.

### Storage Layout & Mounts

-   **OS Root:** `/dev/sda2` (`ext4`, 915G total, 854G available) mounted at `/`.
-   **Media SSD:** `/dev/sdb1` (`exfat`, 1.9T total, 295G available) mounted persistently at `/srv/media`.

### Docker Volume Mappings

-   **Cache:** `/srv/jellyfin/cache` -> `/cache` (Read/Write)
-   **Configuration:** `/srv/jellyfin/config` -> `/config` (Read/Write)
-   **Media Library:** `/srv/media` -> `/media` (Read-Only)

### Container Configuration

-   **Restart Policy:** `unless-stopped` (ensures the container automatically recovers and boots alongside the system after a reboot).

## 8. Connectivity and Service Tests

| Test                  | Command                        | Purpose                         |
| --------------------- | ------------------------------ | ------------------------------- |
| Router connectivity   | `ping -c 4 192.168.1.254`      | Check gateway reachability      |
| Internet connectivity | `ping -c 4 1.1.1.1`            | Test external IP connectivity   |
| DNS resolution        | `getent hosts google.com`      | Check name resolution           |
| SSH login             | `ssh sfmcmullen@192.168.1.222` | Test remote administration      |
| Container status      | `sudo docker ps`               | Check Jellyfin is running       |
| Listening sockets     | `sudo ss -tulpn`               | Inspect local service listeners |

The router, internet, and DNS connectivity tests have previously succeeded. SSH connectivity has also been tested over the home LAN.

A successful ping does not establish that a particular application port is reachable. Test the relevant service directly when troubleshooting.

## 9. Security Considerations

-   Keep SSH key authentication enabled and password-based SSH login disabled.
-   Keep SSH restricted to trusted sources.
-   Protect private SSH keys and never commit them to Git.
-   Review Docker port publishing and firewall rules when adding services.
-   Do not expose SSH or Jellyfin publicly without a deliberate security review.
-   Do not publish passwords, API tokens, private keys, or other secrets in this repository.

## 10. Change Log

Record significant configuration changes and how they were tested.

| Date       | Change                                                        | Verification                                               |
| ---------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
| 2026-10-12 | Enabled UFW                                                   | Inspected firewall status                                  |
| 2026-10-12 | Restricted SSH to the home IPv4 subnet                        | Inspected numbered firewall rules                          |
| 2026-10-12 | Configured SSH key authentication and disabled password login | Checked effective SSH settings and tested a new connection |
| 2026-10-10 | Deployed Jellyfin using Docker                                | Checked container status and local service access          |
