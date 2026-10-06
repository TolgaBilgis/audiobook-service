# Audiobookshelf

Apply `audiobookshelf.yml` with kubectl. This directory is separate from the
portfolio Argo CD application's watched `kubernetes/apps` directory. It is
currently managed manually, not synchronized by Argo CD.

One instance runs on node02. Four local-path volumes hold `/config`, `/metadata`,
`/audiobooks`, and `/podcasts`. Their Retain policy preserves data when claims
are deleted; reclaiming a released volume requires manual intervention.
Requested sizes (5/10/100/20 GiB) are planning values: local-path does not enforce
disk quotas or reserve that space. Monitor the node's free disk space.
Node02 downtime makes this application unavailable. Retention is not a backup:
copy application backups and media to a separate disk or machine.

## First login

From a computer with cluster access:

```sh
kubectl -n audiobookshelf port-forward svc/audiobookshelf 13378:80
```

Open http://localhost:13378 and complete the administrator account setup before
publishing the public hostname. Add an audiobook library using `/audiobooks`.
The web upload feature writes books to this volume. Optional podcasts use
`/podcasts`. Create separate non-admin accounts for friends.

## Cloudflare

Reuse the existing portfolio tunnel. In Cloudflare's dashboard, select the
tunnel, then Routes > Add route > Published application (older dashboard
layouts may call this Published application routes).

- Hostname: `books.tolgabilgis.com`
- Service type: HTTP
- Service URL: `audiobookshelf.audiobookshelf.svc.cluster.local.:80`

Keep the existing portfolio route. The dashboard manages the hostname's DNS.
No router port forwarding or extra cloudflared deployment is required.
The hostname is public and Audiobookshelf handles user authentication.

## Updates and backups

The image is pinned to version 2.37.1. Back up application data before changing
the tag and applying the manifest. Recreate deployment updates stop the old
instance before starting the new one. Application backups do not replace a
separate backup of the media files. No off-node backup schedule is installed.

The trailing dot in the service hostname avoids external DNS search suffix
timeouts observed in this cluster. The absolute hostname returned HTTP 200.

## Torrent downloads

Transmission runs on node02 in the same namespace and mounts the existing
`audiobooks` claim. Apply `transmission.yml` to recreate the deployment.
The image version is pinned; the daemon runs directly as UID/GID 1000 instead
of using the image's root initialization system. Its RPC endpoint binds only
to pod localhost and is accessed through authenticated `kubectl exec`.
There is no public service or Cloudflare route for Transmission.

The executable `../../bin/audiobook-download` is linked into
`/home/tolga/.local/bin/audiobook-download` on this machine:

```sh
audiobook-download 'magnet:?xt=urn:btih:YOUR_HASH'
audiobook-download 'https://example.com/book.torrent'
audiobook-download status
audiobook-download info 1
audiobook-download pause 1
audiobook-download resume 1
```

Quote links to preserve shell characters such as `&`. HTTP(S) links must
return torrent metadata, not an HTML mirror/download page. Downloads start
when queued, and continue after the command exits. Torrents need available
peers or web seeds. Archives are not automatically extracted.

Unfinished files remain in `/audiobooks/.incomplete`, which Audiobookshelf
ignores. Completed files move to `/audiobooks`, where the library watcher
can detect them. Both paths share node02's existing audiobook volume.
Transmission's state is stored on a separate retained 1 GiB claim.
The initial settings ConfigMap is copied only on first startup; persisted
settings thereafter live in `/config/settings.json` (edit only while stopped,
or use transmission-remote for live changes).

Transmission now routes through the Surfshark WireGuard sidecar.
Audiobookshelf and Cloudflare Tunnel keep their original network routes. No automatic router port mappings are enabled. Outgoing peer
connections work without port forwarding, but some swarms may have fewer
reachable peers. Transmission may upload/seed as well as download; use pause
to stop a torrent. No real book downloads are included with this setup.
