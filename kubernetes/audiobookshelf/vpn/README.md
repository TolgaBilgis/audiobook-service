# Surfshark VPN

The canonical VPN deployment is now in `../transmission.yml`. The saved
patch is a reference; normal deployments should apply the complete manifest.
The Surfshark Atlanta configuration is stored in Kubernetes Secret
`surfshark-wireguard`, key `wg0.conf`. Credentials are not stored in Git.
Keep the original WireGuard configuration in secure storage for recovery.

Gluetun v3.41.3 runs as a native sidecar and must be healthy before
Transmission starts. Gluetun manages the pod firewall and DNS; no LAN or
cluster outbound subnet bypasses are configured. IPv4 uses WireGuard and
IPv6 egress is blocked by Gluetun unless explicitly supported/configured.
Transmission stays non-root with all capabilities dropped. Only Gluetun
runs as root with NET_ADMIN and a /dev/net/tun device mount. There is no
host networking. Transmission's storage and completed-download paths are
unchanged. The CLI explicitly selects the Transmission container.

Pod Security admission enforcement for namespace `audiobookshelf` allows
the VPN's capabilities/device; audit and warn stay at restricted. This is
a namespace admission exception, not a privileged-container setting.
Audiobookshelf itself retains its original restricted container settings.

VPN control is bound to pod loopback on port 8000. The unauthenticated role
only permits reading/changing VPN status, for local operational checks.
Other control endpoints remain authenticated. No control port is published.

A VPN startup failure prevents Transmission starting. Gluetun's firewall
blocks ordinary egress when the tunnel stops. For recovery, inspect Gluetun
logs without printing the Secret. Do not remove the sidecar, disable the
firewall, or reapply the old non-VPN manifest as a connectivity workaround.

## Verification — October 5, 2026

Observed Surfshark egress 89.222.103.35 (Atlanta), while Audiobookshelf
retained home egress 104.48.42.254. The public Audiobookshelf site returned
HTTP 200. A controlled VPN stop blocked TCP connections to 1.1.1.1:443 and
8.8.8.8:443, and blocked a direct UDP DNS query to 1.1.1.1:53. The VPN was
then restarted. These checks cover observed IPv4 egress, not all possible
network failure modes.

After the final rollout, a generated test torrent completed into the shared
audiobook volume and its file contents matched. Test data was removed.
IPv6 OUTPUT policy is DROP, with only loopback, established connections,
link-local discovery and the VPN interface allowed.
