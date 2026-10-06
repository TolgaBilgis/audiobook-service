# Validation and known limits

## Observed results

The homelab deployment was checked on October 5, 2026:

- Audiobookshelf started successfully as UID/GID 1000.
- All four persistent volume claims bound to the selected worker.
- The internal service returned HTTP 200 with an absolute DNS hostname.
- The public HTTPS route returned the Audiobookshelf page with HTTP 200.
- Administrator setup and library creation completed.
- A completed audio file was inspected with ffprobe and successfully indexed.

These are point-in-time functional checks, not a load test, durability benchmark
or availability guarantee. The published reference uses a configurable node
label instead of the original fixed worker name and a restricted namespace
policy; deployment to a new cluster still requires validation there.

## Known limits

- The application and media depend on one worker's local disk.
- Local-path requested sizes are not enforced quotas.
- Off-node backups and restore testing remain outstanding.
- The tunnel adds an external dependency for public availability.
- Upstream DNS timeouts were observed in the cluster; an absolute service
  hostname solved the route lookup without resolving all cluster DNS issues.
- There is no automated deployment pipeline or formal security audit.
