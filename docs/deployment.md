# Deployment

## Prerequisites

Use a Kubernetes cluster with Rancher's local-path provisioner already installed.
Select one worker with enough free disk space for the library and metadata.
A cluster administrator must create the StorageClass and namespace.

Label only the intended storage worker `audiobookshelf.storage=local`. The
manifest's selector pins the application to that node; local-path volume
binding records node affinity. Changing a label does not migrate volume data.

The reference namespace enforces the restricted Pod Security profile. Review
existing namespace policy before applying to an established environment:
namespace-level policy changes may affect other workloads. The portable
reference is not a complete export of a particular cluster's admission policy.

## Storage

| Claim | Requested capacity | Container path |
|---|---|---|
| config | 5 GiB | `/config` |
| metadata | 10 GiB | `/metadata` |
| audiobooks | 100 GiB | `/audiobooks` |
| podcasts | 20 GiB | `/podcasts` |

The local-path provisioner does not enforce these sizes as disk quotas.
Monitor actual free space. The Retain policy prevents automatic volume cleanup
after claim deletion but is not a backup. Configuration contains SQLite and
must remain on suitable local storage rather than an NFS/SMB share.

Apply `kubernetes/audiobookshelf.yaml`, wait for readiness, and connect through
port forwarding as shown in the README. Create an administrator and a library
using `/audiobooks`. For other listeners, create non-admin accounts and grant
library access; allow downloads if offline listening is wanted.

## Public route

Use an existing Cloudflare tunnel whose connector can reach cluster services.
In its dashboard, add a published application route:

| Field | Value |
|---|---|
| Hostname | `books.YOUR_DOMAIN` |
| Service type | HTTP |
| Origin | `audiobookshelf.audiobookshelf.svc.cluster.local.:80` |

The trailing dot specifies an absolute DNS name. It avoids unnecessary search
suffix lookups, which caused delays in the observed cluster. HTTPS terminates
at Cloudflare; the connector reaches the ClusterIP service over HTTP inside
the cluster. This does not imply TLS on the internal application hop.

Keep tunnel tokens in a Secret or a separate credential store, outside Git.
The tunnel's deployment, DNS and account configuration are not automated here.
Review the provider's current terms and suitability for your audio traffic.

## Existing installations

Before applying to an existing namespace, inspect current deployments, claims,
node labels and admission policy. Do not recreate claims to change their sizes
or relocate media. Back up data first and plan volume migration explicitly.
The manifest uses pinned application version 2.37.1; verify release notes and
backup compatibility before upgrading.
