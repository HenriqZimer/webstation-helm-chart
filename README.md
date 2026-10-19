# webstation Helm Chart

Helm chart for [romm-webstation](https://github.com/romm-streaming/romm-webstation): the all-in-one
emulation desktop (Selkies) + `romm-broker` used by [RomM](https://romm.app/)'s `protocol: webstation`
emulator streaming. One container serves every emulator to RomM.

See [chart/README.md](chart/README.md) for installation, how to wire it into RomM, and the values.

```bash
helm lint chart
helm template webstation chart
```

Part of the HenriqZimer emulator charts (see also `romm-helm-chart`).
