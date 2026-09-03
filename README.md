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
| `service.type` | `LoadBalancer` or `ClusterIP` | `LoadBalancer` |
| `service.ports.game` / `.query` / `.blobSync` | UDP ports (`GamePort`/`QueryPort`/`BlobSyncPort`) | `8766` / `27016` / `9700` |
| `service.loadBalancerIP` | Pin a specific IP when `service.type` is `LoadBalancer` (e.g. a MetalLB address) | `""` (auto-assigned) |
| `hostNetwork` | Bind `service.ports` directly on the node instead of a pod IP. Needed for LAN clients: the SOTF client broadcasts its "Direct Connect" query to `255.255.255.255` for RFC1918 addresses instead of unicasting to the IP you typed, which a LoadBalancer/MetalLB VIP never sees since it isn't a real host on the broadcast domain. Use `service.type: ClusterIP` alongside this instead of `LoadBalancer` | `false` |
| `dnsPolicy` | Pod DNS policy; only meaningful with `hostNetwork: true`, where it defaults to `ClusterFirstWithHostNet` to keep in-cluster DNS working | `""` (Kubernetes default) |
| `paths.gamePath` / `.userDataDir` | Where the game is installed / its userdata subdirectory; also set as the `GAME_PATH`/`GAME_USERDATA_PATH`/`GAME_CONFIGFILE_PATH` env vars so the volume mount and the server always agree | `/sonsoftheforest` / `userdata` |
| `persistence.enabled` / `.size` / `.storageClassName` | Storage for `paths.gamePath` | `true` / `20Gi` / `""` |
| `persistence.existingClaim` | Reuse an existing PVC instead of creating one | `""` |
| `resources` | Pod resource requests/limits | `{}` (unset) |

See [`values.yaml`](values.yaml) for the full list, including probe tuning, `extraEnv`,
`nodeSelector`, `tolerations`, and `affinity`.

### Per-server settings (name, password, max players, ...)

By default, `ServerName`, `MaxPlayers`, `Password`, `GameMode`, etc. are not exposed as chart
values — the image only generates `userdata/dedicatedserver.cfg` (and `userdata/ownerswhitelist.txt`,
the owner/admin SteamID whitelist) on first install, and otherwise leaves them alone. Edit them
directly on the persistent volume after the first start, then restart the pod:

```console
kubectl exec -it deploy/sotf-sons-of-the-forest -- vi /sonsoftheforest/userdata/dedicatedserver.cfg
kubectl rollout restart deploy/sotf-sons-of-the-forest
```

Alternatively, set `setup.enabled: true` to manage `dedicatedserver.cfg`,
`ownerswhitelist.txt`, and `steam_appid.txt` from chart values instead. An initContainer syncs two
ConfigMaps onto the volume **on every pod (re)start** — one mounted as `paths.userDataDir`
(`dedicatedserver.cfg`, `ownerswhitelist.txt`) and one mounted as the `paths.gamePath` root
(`steam_appid.txt`) — overwriting a file whenever it's missing or differs from its ConfigMap:

| Key | Description | Default |
| --- | --- | --- |
| `setup.enabled` | Sync the files below from two ConfigMaps on every pod start | `false` |
| `setup.serverConfig` | Map of config keys (`ServerName`, `MaxPlayers`, `Password`, `GameMode`, ...) rendered to `dedicatedserver.cfg` (JSON) | see `values.yaml` |
| `setup.ownerSteamIds` | List of SteamIDs rendered to `ownerswhitelist.txt`, one per line | `[]` |
| `setup.steamAppId` | Content of `steam_appid.txt` (shouldn't normally need changing) | `"1326470"` |
| `setup.existingUserDataConfigMap` | Use an existing ConfigMap (keys `dedicatedserver.cfg`, `ownerswhitelist.txt`) instead of templating one | `""` |
| `setup.existingGameDirConfigMap` | Use an existing ConfigMap (key `steam_appid.txt`) instead of templating one | `""` |
| `setup.image` | Image used for the sync initContainer | `busybox:1.36` |

`IpAddress`, `GamePort`, `QueryPort`, and `BlobSyncPort` in the rendered `dedicatedserver.cfg` are
always derived from `service.ports` and can't be overridden via `setup.serverConfig`.

**Once `setup.enabled` is `true`, don't hand-edit these files in-game or on the volume** —
the next pod restart will detect the difference from the ConfigMap and overwrite them.

## Known issue: LAN clients and `service.type: LoadBalancer`

If you run this behind a `LoadBalancer` Service (e.g. a MetalLB VIP) and LAN clients get
**"Game server not found"** on the game's Direct Connect screen — even though the VIP is fully
reachable over raw UDP — this is a known SOTF client quirk, not a bug in the chart, Kubernetes, or
MetalLB.

What we confirmed by packet-capturing both a LAN client and the node during a real deployment
(MetalLB L2 mode, single-node k3s):

- Raw UDP sent to the VIP's `game`/`query`/`blobSync` ports traverses MetalLB's ARP announcement
  and iptables DNAT and lands in the pod correctly — verified end to end with a manual
  `System.Net.Sockets.UdpClient.Send()` from the client plus `tcpdump` on the node and on the
  pod's network namespace. The network path is not the problem.
- The actual game client's "Direct Connect" to that same VIP produced **zero** outbound packets
  from the client machine during the same capture window — it never even attempted to open a
  socket to the address entered.

That's consistent with reports elsewhere that the SOTF client's LAN discovery doesn't unicast its
server-info query to the address you type when it's an RFC1918/private address — instead it
broadcasts to `255.255.255.255` on the query port, and reportedly only unicasts directly to a
public/WAN-routable address. We haven't independently packet-captured that broadcast ourselves,
but it fits what we saw: a `LoadBalancer`/MetalLB VIP is a virtual address answered only via ARP
proxying, not a real host that receives LAN broadcast traffic, so it would never see that query no
matter how reachable it is by direct unicast — while a Docker container published with
`--network host` on an actual LAN host would.

**Fix**: set `hostNetwork: true` (and switch `service.type` to `ClusterIP`, since the ports are
now already exposed on the node — a `LoadBalancer` VIP alongside `hostNetwork` would just
reintroduce the same problem for anyone who connects to the VIP instead of the node). This binds
the game ports directly to the node's real LAN IP, the same way `docker run --network host` is
reachable, so a LAN client's broadcast-to-`255.255.255.255` reaches an actual host on the wire.
You lose the "movable"/multi-node LoadBalancer-IP abstraction, but keep everything else this chart
gives you (PVC, ConfigMaps, restart policy, GitOps, etc.) — and on a single-node cluster there was
nothing to move anyway.

If this server only needs to be reachable from outside your LAN (e.g. port-forwarded to a WAN IP),
`service.type: LoadBalancer` without `hostNetwork` may be sufficient, since a direct-unicast query
to a public IP was reported to behave differently from the LAN broadcast case above — we haven't
verified that ourselves.

## Notes

- **Single instance only**: the chart is a `Deployment` pinned to `replicaCount: 1` with a
  `Recreate` rollout strategy, since the game server is stateful and the volume is mounted
  `ReadWriteOnce`. It cannot be horizontally scaled.
- **First start is slow**: SteamCMD downloads the ~10GB+ dedicated server on first boot. The
  `startupProbe` allows up to ~1 hour for this before Kubernetes considers the pod unhealthy;
  follow progress with `kubectl logs -f deploy/sotf-sons-of-the-forest`.
- **UDP only**: all three ports are UDP, and there is no HTTP endpoint, so this chart deploys no
  `Ingress`, `HorizontalPodAutoscaler`, or Helm test hook.
