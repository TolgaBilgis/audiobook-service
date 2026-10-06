# Audiobookshelf, Transmission and Surfshark handoff

Last updated: October 5, 2026, America/New_York. Values below describe the
setup established in this chat; recheck live state before changing it.

## Purpose and user preferences

Host audiobooks on the homelab for the owner and friends. Public access should
work in a browser or iOS client without requiring friends to install a VPN.
Download torrents from a single CLI command, store completed media in the
Audiobookshelf library, and route only Transmission through Surfshark.
Audiobookshelf, Cloudflare and other workloads must use normal networking.

## Repositories and local files

- Public infrastructure repository: https://github.com/TolgaBilgis/homelab
- Original local checkout: `/home/tolga/homelab`
- This private handoff workspace: `/home/tolga/homelab-private`
- Original manifests: `/home/tolga/homelab/kubernetes/audiobookshelf/`
- Original CLI: `/home/tolga/homelab/bin/audiobook-download`
- Installed CLI link: `/home/tolga/.local/bin/audiobook-download`
- This private repo includes copies of those manifests and the CLI for recovery.
  Reconcile changes with the installed local files; these are not automatically
  synchronized. Do not publish the private handoff to the public repository.
- Existing unrelated changes in the public checkout included a staged rack
  photo and an untracked pipeline image. They were not committed by this work.
- Audiobookshelf and Transmission are manually applied, not managed by Argo CD.
  The portfolio Argo app watches `kubernetes/apps`; these files were deliberately
  placed in the separate `kubernetes/audiobookshelf` directory.

## Cluster and applications

Cluster context: `kubernetes-admin@kubernetes`. Kubernetes version observed:
1.36.4, Ubuntu nodes with containerd and Flannel.

| Node | LAN address | Role |
|---|---|---|
| node01 | 192.168.1.201 | control plane |
| node02 | 192.168.1.202 | worker; audiobook and torrent data |
| node03 | 192.168.1.203 | worker |

Namespace: `audiobookshelf`.

- Deployment `audiobookshelf`: one replica, Recreate updates, pinned to node02.
  Image `ghcr.io/advplyr/audiobookshelf:2.37.1`.
  Container listens on 8080; ClusterIP Service `audiobookshelf` exposes port 80.
  UID/GID 1000, capabilities dropped, RuntimeDefault seccomp, no service-account
  token. Persistent mounts: `/config`, `/metadata`, `/audiobooks`, `/podcasts`.
- Deployment `transmission`: one replica, Recreate updates, pinned to node02.
  Image `lscr.io/linuxserver/transmission:4.1.3-r0-ls363`.
  Runs the packaged daemon directly as UID/GID 1000, bypassing s6 root startup.
  State in `/config`; audiobook PVC shared with Audiobookshelf.
- Native sidecar `gluetun` in Transmission's pod:
  `qmcgaw/gluetun:v3.41.3`, initContainer with `restartPolicy: Always`.
  Startup probe must pass before Transmission starts.

## Persistent data

StorageClass `audiobookshelf-local-retain` uses `rancher.io/local-path`,
WaitForFirstConsumer and Retain. Claims:

| Claim | Requested size | Purpose |
|---|---|---|
| config | 5 GiB | SQLite DB, users and settings |
| metadata | 10 GiB | covers, metadata and application backups |
| audiobooks | 100 GiB | media, shared with Transmission |
| podcasts | 20 GiB | optional podcasts |
| transmission-config | 1 GiB | torrent queue, resume state and settings |

These are planning sizes: local-path does not reserve/enforce those quotas.
Node02 had about 225 GB free before media was added. Recheck before large imports.
Data is not replicated. Node02 downtime means application downtime. Retain
preserves data after claim deletion, but recovery/rebinding is manual.

Actual audiobook directory on node02:

```text
/opt/local-path-provisioner/pvc-c10c50e0-f809-4078-b865-36f45d8b11e3_audiobookshelf_audiobooks
```

Inside both app and downloader, this is `/audiobooks`. In-progress torrents
live in `/audiobooks/.incomplete`; Audiobookshelf ignores hidden directories.
Completed torrents move into `/audiobooks` and its watcher imports them.
Use separate folders per book, ideally `Author/Book Title/files`.

An initial manually copied book was scanned before copying finished, producing
an invalid-audio error. Once complete, ffprobe could read it; moving it into
its own book folder triggered a successful scan. Check file completeness,
permissions, app logs and library path before assuming an app/client failure.

## Public access and users

URL: https://books.tolgabilgis.com
Portfolio remains at https://tolgabilgis.com.

Existing `cloudflared` deployment in namespace `portfolio` has two replicas.
The existing Cloudflare tunnel was reused; no new router port forwarding.
Dashboard route:

- Public hostname: `books.tolgabilgis.com`
- Service type: HTTP
- Origin: `audiobookshelf.audiobookshelf.svc.cluster.local.:80`

The trailing DNS dot is intentional: the normal FQDN triggered external search
suffix lookup timeouts. The absolute name returned HTTP 200. Cluster DNS had
upstream timeout/IPv6 reachability errors; this broader DNS issue was not fixed.
Cloudflare routes/DNS are dashboard-managed, not captured in these manifests.
Cloudflare's tunnel token remains in the existing `portfolio/cloudflared-token`
Secret; do not print/export it into this repository.

The owner completed initial administrator setup. Users are created in the web
UI: Settings -> Users -> Add User. Friends should use non-admin accounts with
library access; download permission is needed for offline use. Each has their
own listening progress. Passwords are not documented here.

Prologue was recommended for iOS after checking current App Store ratings;
AudioBooth and plappa are alternatives. Set the server to the public HTTPS URL
and sign in with the individual Audiobookshelf account.

Cloudflare audio-delivery terms were discussed. The user explicitly chose to
try the tunnel and switch later if needed. A future direct ingress replacement
can preserve the public hostname, library and accounts.

## Downloader CLI

Run on the machine with the installed CLI and configured Kubernetes access:

```sh
audiobook-download 'magnet:?xt=urn:btih:YOUR_HASH'
audiobook-download 'https://example.com/book.torrent'
audiobook-download status
audiobook-download info 2
audiobook-download pause 2
audiobook-download resume 2
```

Quote links. The command accepts magnet links or direct HTTP(S) torrent URLs,
not arbitrary mirror/download HTML pages. It uses argument arrays without
shell interpolation, pins the Kubernetes context, and explicitly execs into
container `transmission`. Downloads continue after the terminal closes.
No public Transmission Service or web route exists. RPC listens only on pod
loopback port 9091. No archive extraction or media acquisition sources are set up.
Transmission also uploads/seeds; pause stops that torrent. Limits were unlimited
at verification. The active queue is transient; inspect rather than relying on
old IDs/titles. Do not delete active downloads during routine troubleshooting.

## Surfshark isolation and credentials

The supplied Surfshark Atlanta manual WireGuard config was validated and
imported into Kubernetes Secret `audiobookshelf/surfshark-wireguard`, key
`wg0.conf`. No key is committed. Obtain a fresh manual config from the owner
or securely recover the existing Secret if a rebuild is necessary.
Only recognized WireGuard fields were imported; no config hooks were executed.
Custom Gluetun configuration requires an IP endpoint; hostnames were resolved
before import. Current endpoint: `89.222.103.34:51820`.

VPN settings: custom provider, WireGuard kernelspace, firewall on. Pod DNS is
127.0.0.1 through Gluetun. Transmission shares only its own pod's network with
Gluetun. No host networking, no manual outbound subnet bypasses. Gluetun still
has automatic rules for pod-local subnet traffic; do not describe this as zero
local-network access. IPv6 output uses a DROP policy with narrow exceptions.

Gluetun runs as root with NET_ADMIN and a `/dev/net/tun` hostPath. It is not a
privileged container. PUID/PGID=0 and a startup chmod on `/tmp/gluetun` prevent
public-IP status-file permission/chown errors without adding DAC_OVERRIDE.
Transmission and Audiobookshelf retain their non-root security contexts.

Namespace Pod Security enforce is `privileged` to admit the root/capability/
device requirements; audit and warn remain `restricted`. This relaxes admission
for the namespace, not the existing app container settings. Do not change it
back without accommodating the VPN pod. Future hardening could isolate VPN
workloads in another namespace, but requires planning shared local storage.

Gluetun control server binds to 127.0.0.1:8000. The role permits only GET/PUT
`/v1/vpn/status` without auth from the pod; other endpoints remain protected.
No control port is exposed publicly. A startup health gate plus Gluetun's
firewall prevents ordinary internet fallback when the tunnel is down.
Never disable the firewall/remove the sidecar to fix slow downloads.

## Verification and troubleshooting

Observed on October 5:

- Transmission VPN egress: `89.222.103.35`, Atlanta.
- Audiobookshelf/home egress: `104.48.42.254` (may change).
- Both app workloads ready; public website returned HTTP 200.
- Controlled VPN stop blocked TCP to 1.1.1.1:443 and 8.8.8.8:443 and an external
  UDP DNS query to 1.1.1.1:53; VPN then restarted successfully.
- Generated local test torrent completed into the shared volume, verified its
  contents and removed its own fixture. This tests completion/storage handling;
  VPN throughput was separately tested with an external HTTP download.
- External 10 MB download through VPN measured about 22.5 MB/s. One slow torrent
  had three connected peers and intermittent transfers with unlimited limits.
  That supports peer availability/upload speed as the likely bottleneck, not
  a general VPN bandwidth cap. It does not guarantee every swarm's throughput.
- A later check confirmed two active torrents still used the VPN IP and firewall.

Useful checks (avoid printing Secrets or full configuration credentials):

```sh
kubectl -n audiobookshelf get pods,pvc -o wide
audiobook-download status
kubectl -n audiobookshelf logs deployment/transmission -c gluetun --tail=25
kubectl -n audiobookshelf exec deployment/transmission -c gluetun -- /gluetun-entrypoint healthcheck
kubectl -n audiobookshelf exec deployment/transmission -c transmission -- curl -fsS --max-time 15 https://api.ipify.org
kubectl -n audiobookshelf exec deployment/transmission -c gluetun -- iptables -S OUTPUT
kubectl -n audiobookshelf logs deployment/audiobookshelf --tail=50
curl -I https://books.tolgabilgis.com
```

Public IP alone is a point-in-time observation. For VPN changes, additionally
verify routing/firewall and a controlled tunnel-down test, then restore the VPN.
Coordinate disruptive tests if users have active downloads. Native sidecar
shutdown and readiness can make rollouts take around a minute.

## Recovery, deployment and remaining work

Do not blindly apply manifests in another cluster or recreate claims. Confirm
node02, existing PVC binding, retained PV paths and current storage state first.
For an existing installation the full manifests are `audiobookshelf.yml` and
`transmission.yml`; the VPN patch is reference material, not the canonical
standalone deployment. Secret import and Cloudflare dashboard configuration
are separate prerequisites. Restore credentials before starting Transmission.

Outstanding:

1. No off-node backup or tested restore exists. Back up database/application
   data and media separately; local application backups do not include media.
2. Local node storage is a single point of failure; quotas are not enforced.
3. Fix broader cluster upstream DNS reliability separately if requested.
4. GitOps integration is not implemented. Avoid placing these resources in the
   existing portfolio Argo app's watched directory without redesigning scope.
5. This handoff and manifests must be kept current after later changes.

Future chats should open this document and inspect the live cluster before
acting. Sharing a repository does not automatically inject its contents into
all chats: point the next chat at this private repo/handoff path.

## October 6 follow-up

The user created the private repository `TolgaBilgis/audiobook-service` for
this handoff and deployment snapshot. The local staging directory remains
`/home/tolga/homelab-private`.

Pending request: build an interactive search/select/download command using
https://github.com/JamesRy96/audiobookbay-automated. Only its README/source
were inspected and a temporary clone was made at `/tmp/audiobookbay-api-review`.
No search command or API service was installed. The existing
`audiobook-download` accepts supplied links only. Do not report the search
integration as complete; inspect upstream endpoints and resume implementation
when the user returns to this request.
