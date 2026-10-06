# Follow-up work

Read docs/audiobookshelf-handoff.md before changing this setup. Inspect live
state and repository status; the document is a dated snapshot, not live state.

Keep this repository private. Never commit credentials, WireGuard configs,
kubeconfigs, databases, media files, or Kubernetes Secret values.

Transmission must remain behind Gluetun/Surfshark with its firewall enabled.
Do not remove the sidecar or kill switch as a connectivity workaround.
Audiobookshelf and the portfolio Cloudflare tunnel must remain outside the VPN.
Preserve existing media and persistent volumes. There is no off-node backup.

The original local deployment files live in /home/tolga/homelab, whose GitHub
remote is PUBLIC. Do not push this private handoff to that remote. Its unrelated
staged/modified files belong to other work; do not include them in commits.

Use explicit container selection (-c transmission or -c gluetun) for pod exec.
Verify hostname access, app readiness, VPN egress, and relevant storage behavior
after changes. Record consequential changes and verification in the handoff.
