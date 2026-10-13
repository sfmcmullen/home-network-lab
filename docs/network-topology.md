# Network Topology and Configuration

## 1. Overview

The lab consists of a home LAN, a Windows administration PC, an Ubuntu Server host, and a Docker container running Jellyfin.

The server is managed remotely over SSH using key-based authentication. UFW restricts inbound access at the host level. Docker publishes Jellyfin on the server's Wi-Fi address for local-network access.

## 2. Network Topology

```text
                         Internet
                             |
                    Public IPv4 Address
                             |
                       Home Router
                      192.168.1.254
                             |
                Port Forwarding: TCP 80 / 443
                             |
                 Home LAN: 192.168.1.0/24
                             |
                      Ubuntu Server
                      mcmullen-lab
                     192.168.1.222
                             |
                       UFW Firewall
                             |
                     Caddy (systemd)
                       TCP 80 / 443
                             |
                      Reverse Proxy
                    Upstream Endpoint
                   192.168.1.222:8096
                             |
                       Docker Engine
                             |
                    Jellyfin Container
                        TCP 8096

DNS:
  Cloudflare DNS-only A record
  jellyfin.sfmcmullen.com -> Public IPv4 address

LOCAL ADMINISTRATION (HOME NETWORK ONLY)

  Windows PC                                  Ubuntu Server
  192.168.1.85                                mcmullen-lab
       |                                      192.168.1.222
       +------------- SSH / TCP 22 ---------------->
                     SSH key required
```

The Windows PC administers the server over SSH. Devices on the home LAN can access Jellyfin through the server's published endpoint, `192.168.1.222:8096`.

The Docker container uses a Docker-managed network. Its container IP is not the same as the server's LAN address.

## 3. IP Addressing

| Service or resource      | Address or hostname       | Purpose                         |
| ------------------------ | ------------------------- | ------------------------------- |
| Jellyfin public endpoint | `jellyfin.sfmcmullen.com` | External HTTPS access           |
| Caddy HTTP listener      | `192.168.1.222:80`        | HTTP and ACME challenge traffic |
| Caddy HTTPS listener     | `192.168.1.222:443`       | HTTPS reverse proxy             |
| Jellyfin upstream        | `192.168.1.222:8096`      | Internal application endpoint   |
| Home router              | `192.168.1.254`           | Default gateway                 |
| Windows PC               | `192.168.1.85`            | Administration client           |
| Ubuntu server Wi-Fi      | `192.168.1.222`           | Host LAN address                |

The home LAN uses the IPv4 subnet `192.168.1.0/24`, equivalent to subnet mask `255.255.255.0`.

The server's Wi-Fi interface is `wlx28ee521c9351`.

The public hostname `jellyfin.sfmcmullen.com` provides external HTTPS access through Caddy, which listens on ports `80` and `443` on the Ubuntu server's LAN address. Caddy forwards incoming HTTPS requests to the Jellyfin application at `192.168.1.222:8096`.

The Jellyfin upstream address uses the Ubuntu server's LAN IP and published port rather than the container's internal Docker bridge IP. The container IP, previously observed as `172.18.0.2`, may change if the container or Docker network is recreated. Using the public hostname for external access and the host address and published port for the upstream connection avoids relying on the container's potentially changing IP address.

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

Public Jellyfin access is intended to pass through Caddy using the hostname `jellyfin.sfmcmullen.com`. The home router should forward incoming TCP ports `80` and `443` to the Ubuntu server at `192.168.1.222`. Caddy handles HTTP redirects, certificate validation, and HTTPS reverse proxying to the internal Jellyfin application endpoint.

SSH must remain restricted to the home IPv4 subnet, `192.168.1.0/24`. The Jellyfin port `8096` must not be forwarded directly through the router. Other inbound traffic should remain denied unless explicitly required.

The intended inbound firewall and router policy is:

| Traffic               | Intended policy                                         |
| --------------------- | ------------------------------------------------------- |
| SSH, TCP 22           | Allow from `192.168.1.0/24` only                        |
| HTTP, TCP 80          | Allow to Caddy for redirects and certificate validation |
| HTTPS, TCP 443        | Allow to Caddy                                          |
| Jellyfin, TCP 8096    | Keep unforwarded at the router                          |
| Other inbound traffic | Deny unless explicitly required                         |

Inspect the current UFW rules:

```bash
sudo ufw status verbose
sudo ufw status numbered
```

These commands show the active UFW policy and numbered rules. Verify that SSH is restricted to `192.168.1.0/24` and that the rules for HTTP and HTTPS match the intended Caddy configuration.

Docker manages its own networking and may create firewall rules for published container ports. Therefore, UFW rules alone should not be treated as proof that a Docker-published service is inaccessible from other interfaces or networks. Confirm the Docker port-publishing configuration and relevant host firewall rules, and ensure that TCP port `8096` is not exposed through router port forwarding.

UFW's IPv6 support is enabled, but the router's IPv6 firewall behaviour has not yet been verified. Do not assume the server is protected from unsolicited inbound IPv6 traffic until the router's IPv6 firewall and relevant server rules have been checked.

The intended public access path is through Caddy on TCP ports `80` and `443`, with Jellyfin remaining behind the reverse proxy rather than being exposed directly to the internet.

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

## 7. Caddy Reverse Proxy and Public HTTPS

Caddy runs directly on the Ubuntu host as a systemd service. It
handles HTTPS connections and reverse-proxies requests to the
Jellyfin service running in Docker.

### Domain and DNS

-   **Public hostname:** `jellyfin.sfmcmullen.com`
-   **DNS provider:** Cloudflare
-   **DNS record:** A
-   **Proxy mode:** DNS-only (grey cloud)
-   **Destination:** Home public IPv4 address

The public IPv4 address is deliberately not recorded in this
repository. If the address changes, the Cloudflare DNS record
must be updated.

### Caddy Configuration

The Caddy configuration file is located at:

`/etc/caddy/Caddyfile`

The current configuration is:

```caddyfile
jellyfin.sfmcmullen.com {
    reverse_proxy 192.168.1.222:8096
}
```

This configuration tells Caddy to accept requests for
`jellyfin.sfmcmullen.com` and forward them to Jellyfin at
`192.168.1.222:8096`.

Caddy manages HTTPS certificates automatically using Let's
Encrypt. On 13 October 2026, the Caddy logs confirmed successful
TLS-ALPN validation and certificate issuance for
`jellyfin.sfmcmullen.com`.

### Useful Commands

Validate the Caddy configuration:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile
```

Check the Caddy service status:

```bash
sudo systemctl status caddy --no-pager
```

View recent Caddy logs:

```bash
sudo journalctl -u caddy --since "15 minutes ago" --no-pager
```

Check connectivity to the Jellyfin upstream:

```bash
curl -I http://192.168.1.222:8096
```

### Network and Security

The intended public entry points are TCP ports 80 and 443,
forwarded by the router to the Ubuntu server at
`192.168.1.222`.

-   TCP 80 supports HTTP-to-HTTPS redirects and certificate
    validation.
-   TCP 443 serves the public HTTPS endpoint.
-   TCP 22 remains restricted to the trusted home IPv4 subnet.
-   TCP 8096 must not be forwarded directly through the router.

Cloudflare DNS-only mode resolves the hostname to the home
public IPv4 address without proxying HTTP traffic through
Cloudflare.

UFW is enabled with IPv6 support, but the router's IPv6
firewall behaviour still needs verification. The host firewall
and router configuration should be checked to ensure that
unwanted inbound connections are blocked.

### External Verification

From a device outside the home network, open:

`https://jellyfin.sfmcmullen.com`

Verify that:

1. The domain resolves to the intended public IPv4 address.
2. The browser reports a valid HTTPS certificate.
3. The Jellyfin login page loads successfully.
4. A media file plays successfully over the remote connection.

Successful certificate issuance alone does not prove that
every external connection and playback path works.

Record the results of these tests in the change log once
verification is complete.

## 8. Storage and Persistence

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

## 9. Connectivity and Service Tests

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

## 10. Media Drive Maintenance

### Configuration

The server uses an external 2 TB media drive with the following configuration:

| Property                | Value            |
| ----------------------- | ---------------- |
| Filesystem              | exFAT            |
| Filesystem UUID         | `2451-AAE4`      |
| Mount point             | `/srv/media`     |
| Jellyfin container path | `/media`         |
| Mount configuration     | `/etc/fstab`     |
| Access mode             | Read-only (`ro`) |

The drive is identified by its filesystem UUID in `/etc/fstab`, rather than relying on a device name such as `/dev/sdc1`. This allows the drive to be mounted consistently even if its device name changes after reconnection.

The `nofail` option allows Ubuntu to boot when the external drive is unavailable. However, the drive does not necessarily remount automatically when physically reconnected while the server is running.

### Safely Removing the Drive

**Never unplug the drive while Jellyfin is running or while the filesystem is mounted.** Doing so can interrupt media access and risk filesystem damage.

1. Stop the Jellyfin container:

    ```bash
    sudo docker stop jellyfin
    ```

2. Unmount the media drive:

    ```bash
    sudo umount /srv/media
    ```

3. Confirm that the filesystem is no longer mounted:

    ```bash
    findmnt /srv/media
    ```

    No output should be returned for the mount point.

4. Only unplug the drive after the unmount command succeeds and the filesystem is confirmed to be unmounted.

If the unmount operation reports that the target is busy, do not force it. Check which processes are using the mount:

```bash
sudo fuser -vm /srv/media
```

Resolve any remaining access before attempting to unmount again.

### Reconnecting the Drive

1. Reconnect the drive to the server.

2. Confirm that Ubuntu detects it and check its filesystem UUID:

    ```bash
    lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS
    ```

    Verify that the expected UUID is `2451-AAE4` and the filesystem is exFAT. Do not assume the device will retain the same `/dev/sdX` name.

3. Check the filesystem without modifying it:

    ```bash
    sudo fsck.exfat -n /dev/disk/by-uuid/2451-AAE4
    ```

    If the check reports filesystem errors, investigate before continuing. Do not attempt repairs without understanding the reported errors.

4. Mount the drive using the existing `/etc/fstab` configuration:

    ```bash
    sudo mount /srv/media
    ```

5. Verify that the correct filesystem is mounted:

    ```bash
    findmnt /srv/media
    ```

    Confirm that the source corresponds to the expected drive and the filesystem is exFAT.

6. Check that the media files are accessible:

    ```bash
    ls -lah /srv/media
    ```

7. Restart Jellyfin only after confirming that the drive is mounted correctly:

    ```bash
    sudo docker start jellyfin
    ```

8. Verify that the container is running:

    ```bash
    sudo docker ps
    ```

9. Open Jellyfin on the local network and confirm that the libraries and media are accessible.

## 11. Security Considerations

-   Keep SSH key authentication enabled and password-based SSH login disabled.
-   Keep SSH restricted to trusted sources.
-   Protect private SSH keys and never commit them to Git.
-   Review Docker port publishing and firewall rules when adding services.
-   Do not expose SSH or Jellyfin publicly without a deliberate security review.
-   Do not publish passwords, API tokens, private keys, or other secrets in this repository.

## 12. Change Log

Record significant configuration changes and how they were tested.

| Date       | Change                                                        | Verification                                               |
| ---------- | ------------------------------------------------------------- | ---------------------------------------------------------- |
| 2026-10-12 | Enabled UFW                                                   | Inspected firewall status                                  |
| 2026-10-12 | Restricted SSH to the home IPv4 subnet                        | Inspected numbered firewall rules                          |
| 2026-10-12 | Configured SSH key authentication and disabled password login | Checked effective SSH settings and tested a new connection |
| 2026-10-10 | Deployed Jellyfin using Docker                                | Checked container status and local service access          |

```

```
