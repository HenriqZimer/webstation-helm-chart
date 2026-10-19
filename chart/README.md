# webstation Helm Chart

Runs [romm-webstation](https://github.com/romm-streaming/romm-webstation) - one container with a
Selkies WebRTC desktop, **every emulator** (PCSX2, Dolphin, RPCS3, Eden, Cemu, xemu, shadPS4, Azahar,
Flycast, PPSSPP, RetroArch, ...) and the `romm-broker` that [RomM](https://romm.app/) drives over REST.
It is the backend for RomM's
[`protocol: webstation`](https://docs.romm.app/latest/using/emulator-streaming/) emulator streaming
and replaces the per-emulator charts + broker mods.

- One container = **one session at a time**, for any platform. Install the chart twice (two releases)
  to get a pool; RomM hands out the first free container.
- Image is **amd64 only**, Wayland only, meant to run **with a GPU**.

## Install

```bash
helm repo add webstation-helm-chart https://henriqzimer.github.io/webstation-helm-chart
helm install webstation webstation-helm-chart/webstation \
  --set broker.secret="$(openssl rand -hex 32)" \
  --set persistence.library.existingClaim=romm-library \
  --set ingress.enabled=true \
  --set 'ingress.hosts[0].host=romm.example.com' \
  --set 'ingress.hosts[0].paths[0].path=/streaming' \
  --set 'ingress.hosts[0].paths[0].pathType=Prefix' \
  --set 'ingress.tls[0].secretName=romm-tls' \
  --set 'ingress.tls[0].hosts[0]=romm.example.com'
```

## Wiring it into RomM

1. **Same library, same path.** RomM sends ROM paths under `/romm/library`. Point
   `persistence.library.existingClaim` at the PVC RomM uses (RomM chart: `<release>-library`, or your
   NFS/TrueNAS claim). It must be ReadWriteMany unless both pods run on the same node.
2. **Same secret.** `broker.secret` here == `STREAMING_BROKER_SECRET` in RomM
   (`secrets.data` of the romm chart). *Alternative:* put `broker_secret` on the container entry in
   `config.yml` and leave the env var unset - the two cannot be mixed (the env var wins whenever set).
3. **HTTPS for the browser.** Selkies WebRTC needs a secure context. The easiest setup is the same host
   as RomM with the path `/streaming` (this chart's Ingress; controllers merge rules per host and the
   longer path wins). `env.SUBFOLDER` (`/streaming/`), the ingress path and `subfolder` in RomM's
   `config.yml` must all agree.
4. **RomM `config.yml`** (`romm.config.data` in the romm chart), RomM >= 5.3:

```yaml
streaming:
  enabled: true
  containers:
    - protocol: webstation
      host: https://romm.example.com/streaming     # or just "/streaming" when on RomM's own origin
      subfolder: /streaming
      broker_host: http://webstation.<namespace>.svc.cluster.local:3000   # server-to-server, plain HTTP is fine
      label: Emulation station
      platforms:
        ps2:
          emulator: pcsx2
          memory_card_sync: true
        ngc: dolphin
        snes: retroarch
        gba: "retroarch:gpsp"
```

   `host` may be a bare path only if you also set `broker_host`. `broker_host` is the identity of a
   pooled container, so two releases serving the same platform need different ones.
5. **First run.** Open `https://<host>/streaming/` (or a *desktop session* from RomM as admin) and set up
   each emulator's BIOS / firmware / keys once. They live in `/config` (`persistence.config`).
   A pre-filled BIOS folder can be mounted with `extraVolumes`/`extraVolumeMounts`.

## Values

| Key | Default | Description |
|---|---|---|
| `image.repository` | `ghcr.io/romm-streaming/romm-webstation` | amd64 only |
| `image.tag` | `v0.15.1` | romm-broker release |
| `broker.secret` | placeholder | `BROKER_SECRET`; **change it** |
| `broker.existingSecret` / `existingSecretKey` | `""` / `BROKER_SECRET` | use a pre-created Secret |
| `env.PUID` / `env.PGID` / `env.TZ` | `1000` / `1000` / `Etc/UTC` | |
| `env.SUBFOLDER` | `/streaming/` | URL prefix of UI + broker API |
| `env.*` | | any broker/Selkies variable (`DRINODE`, `ROM_ROOT`, `EDEN_LEGACY`, `SELKIES_*`, ...) |
| `service.httpPort` / `httpsPort` | `3000` / `3001` | broker + UI are on the HTTP port |
| `ingress.*` | disabled | see above |
| `persistence.config.*` | PVC 20Gi RWO | `/config` (settings, BIOS, saves) |
| `persistence.library.*` | PVC 100Gi RWX, `readOnly: true` | `/romm/library`; use `existingClaim` |
| `shmSize` | `2Gi` | `/dev/shm` (JIT emulators crash with the default) |
| `gpu.enabled` / `gpu.vendor` | `true` / `intel-amd` | or `nvidia` (device plugin + host driver 580+) |
| `gpu.supplementalGroups` | `[]` | host `render`/`video` GIDs |
| `seccompUnconfined` | `false` | only if an emulator JIT crashes (SIGBUS/SIGSEGV) |
| `nodeSelector` | `kubernetes.io/arch: amd64` | |
| `gpu.nvidia.count` | `1` | `0` = do not request `nvidia.com/gpu` (share the GPU with other pods) |
| `resources` | 1 CPU / 2Gi req, 6 CPU / 10Gi limit | |

See `values.yaml` for the full list. Multi-GPU hosts: set `env.DRINODE` and `env.DRI_NODE`
(e.g. `/dev/dri/renderD129`). PCSX2 "Failed to initialize GS" on AMD iGPUs: switch its renderer to OpenGL.
