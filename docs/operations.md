# Operations runbook

## Check service health

```sh
kubectl -n audiobookshelf get pods,pvc -o wide
kubectl -n audiobookshelf logs deployment/audiobookshelf --tail=50
kubectl -n audiobookshelf get service audiobookshelf
kubectl -n audiobookshelf port-forward svc/audiobookshelf 13378:80
```

Check local access first, then the public HTTPS hostname. If local access
works but public access fails, inspect tunnel connectivity, route configuration
and DNS. Check both HTTP responses and the application's logs; a TCP readiness
probe alone does not prove successful playback or database operation.

## Add media

Upload through the web interface or copy files into the audiobook volume.
Use a folder per book, for example `Author/Title/book.m4b`. Allow copying to
finish before scanning. If a book is missing, confirm the `/audiobooks` library
path, read permissions, file completeness and scanner logs, then rescan.

In the observed deployment, a scan encountered an incomplete file during copy.
The completed file passed ffprobe inspection, and moving it into a dedicated
book folder triggered a successful scan. Preserve source files while diagnosing
imports; do not use deletion as the first troubleshooting step.

## Locate the host directory

```sh
kubectl -n audiobookshelf get pvc audiobooks -o jsonpath='{.spec.volumeName}'
# Substitute the returned name below.
kubectl get pv VOLUME_NAME -o yaml
```

Read the volume's hostPath/local path and node affinity. The container path
`/audiobooks` is not necessarily the same as the host's filesystem path.

## Upgrades

Back up application data and media to separate storage. Review upstream release
notes, change the image version, apply the manifest and watch rollout status.
Check login, library contents, streaming and permissions afterward. `Recreate`
updates briefly stop the application. Image rollback may not reverse database
migrations; preserve a compatible pre-upgrade backup.

## Backup and recovery

Back up `/config` consistently with SQLite, along with `/metadata` and the
media volumes. A safe maintenance approach is to stop the app while copying
its database. Application backups do not replace a separate media backup.
No off-node backup schedule or restore drill is supplied by this project.

If the worker is unavailable, local data cannot automatically follow the pod
to another worker. Restore or migrate the data and configure suitable volume
binding before rescheduling. If a claim was deleted, Retain leaves a released
volume requiring administrator intervention. Do not delete or overwrite the
underlying directory during recovery.
