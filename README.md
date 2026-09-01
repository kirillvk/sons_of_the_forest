# sons-of-the-forest

A Helm chart to deploy a [Sons of the Forest](https://www.sonsoftheforestgame.com/) dedicated
server on Kubernetes, using the
[jammsen/docker-sons-of-the-forest-dedicated-server](https://github.com/jammsen/docker-sons-of-the-forest-dedicated-server)
image (game server running under Wine, since it has no native Linux build).

## Prerequisites

- Kubernetes 1.23+
- Helm 3.8+
- A default `StorageClass` (or set `persistence.storageClassName`) able to provision at least
  ~20Gi `ReadWriteOnce` storage
- Cluster/CNI support for the `service.type` you pick (`LoadBalancer` needs an external
  load-balancer implementation such as MetalLB on bare-metal clusters)
- Do **not** apply a restricted Pod Security Standard/policy to this workload — the container's
  entrypoint must start as root to `chown` the game/wine data and `PUID`/`PGID`-drop to an
  unprivileged user

## Installing

```console
helm install sotf ./sons-of-the-forest
```

This deploys the server with default settings (see [Values](#values) below). Check
`helm install`'s `NOTES` output for how to get the connect address and follow install/update
progress.

## Uninstalling

```console
helm uninstall sotf
```

This does not delete the `PersistentVolumeClaim` (savegames, install, wine prefix) created by
the chart — remove it manually if you want a completely clean slate:

```console
kubectl delete pvc sotf-sons-of-the-forest
```

## Configuration

Common settings to override via `--set` or a values file:

| Key | Description | Default |
| --- | --- | --- |
| `image.repository` / `image.tag` | Image to deploy | `jammsen/sons-of-the-forest-dedicated-server` / `latest` |
| `env.PUID` / `env.PGID` | Host uid/gid the process runs as after startup | `1000` / `1000` |
| `env.TIMEZONE` | Container timezone | `Europe/Berlin` |
| `env.ALWAYS_UPDATE_ON_START` | Run a SteamCMD update on every pod start | `true` |
| `service.type` | `LoadBalancer`, `NodePort`, or `ClusterIP` | `LoadBalancer` |
| `service.ports.game` / `.query` / `.blobSync` | UDP ports (`GamePort`/`QueryPort`/`BlobSyncPort`) | `8766` / `27016` / `9700` |
| `persistence.enabled` / `.size` / `.storageClassName` | Storage for `/sonsoftheforest` | `true` / `20Gi` / `""` |
| `persistence.existingClaim` | Reuse an existing PVC instead of creating one | `""` |
| `resources` | Pod resource requests/limits | `{}` (unset) |

See [`values.yaml`](values.yaml) for the full list, including probe tuning, `extraEnv`,
`nodeSelector`, `tolerations`, and `affinity`.

### Per-server settings (name, password, max players, ...)

Settings such as `ServerName`, `MaxPlayers`, `Password`, `GameMode`, etc. are not exposed as chart
values — the image only generates `userdata/dedicatedserver.cfg` on first install and otherwise
leaves it alone (so in-game changes aren't clobbered on restart). Edit it directly on the
persistent volume after the first start, then restart the pod:

```console
kubectl exec -it deploy/sotf-sons-of-the-forest -- vi /sonsoftheforest/userdata/dedicatedserver.cfg
kubectl rollout restart deploy/sotf-sons-of-the-forest
```

## Notes

- **Single instance only**: the chart is a `Deployment` pinned to `replicaCount: 1` with a
  `Recreate` rollout strategy, since the game server is stateful and the volume is mounted
  `ReadWriteOnce`. It cannot be horizontally scaled.
- **First start is slow**: SteamCMD downloads the ~10GB+ dedicated server on first boot. The
  `startupProbe` allows up to ~1 hour for this before Kubernetes considers the pod unhealthy;
  follow progress with `kubectl logs -f deploy/sotf-sons-of-the-forest`.
- **UDP only**: all three ports are UDP, and there is no HTTP endpoint, so this chart deploys no
  `Ingress`, `HorizontalPodAutoscaler`, or Helm test hook.
