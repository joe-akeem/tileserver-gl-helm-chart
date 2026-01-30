# tileserver-gl-helm-chart

A Helm chart for deploying [tileserver-gl](https://github.com/maptiler/tileserver-gl) to Kubernetes.

This README describes how to install and use the chart in its current state. It focuses on the most common scenarios, explains the defaults, and shows how to provide your own MBTiles data.

## Prerequisites
- Kubernetes cluster (v1.23+ recommended)
- Helm v3
- A way to provide MBTiles to tileserver-gl (see "Providing map data (MBTiles)")

## What this chart does (current state)
- Deploys a single tileserver-gl `Deployment` with a `Service`
- Exposes port 8080 inside the cluster (`ClusterIP` by default)
- Optional `Ingress` (disabled by default)
- Optional `Gateway API` `HTTPRoute` (disabled by default)
- Optional `HorizontalPodAutoscaler` (disabled by default)
- Allows passing arbitrary container `args` to tileserver-gl
- Allows attaching custom `volumes`/`volumeMounts` so you can provide data

Note: The default `args` reference a sample file `zurich_switzerland.mbtiles`, which is not bundled in the image. You must provide your own MBTiles file and update the args accordingly.

## Quick start (local directory install)
If you have this repository locally, you can install the chart directly from the directory.

- Create a namespace (optional):
  - `kubectl create namespace maps`
- Install with default values (will run, but will 404 unless you provide MBTiles):
  - `helm install my-tiles ./ -n maps`
- Port-forward to test locally:
  - `kubectl -n maps port-forward svc/my-tiles-tileserver-gl-helm-chart 8080:8080`
  - Open http://localhost:8080

The server will serve tiles from the Zurich area per default. To make serve the desired tiles, you need to provide MBTiles (see below) and set the correct `args`.

## Providing map data (MBTiles)
The chart does not ship data. To use your own `.mbtiles` file mount a PersistentVolume (PVC):

- Create a PVC that contains or is attached to storage with your `.mbtiles` file.
- Reference it using `values.volumes` and mount it with `values.volumeMounts`.

Example snippet for PVC:

```
volumes:
  - name: tiles
    persistentVolumeClaim:
      claimName: my-tiles-pvc

volumeMounts:
  - name: tiles
    mountPath: /data

args:
  - "--file"
  - "/data/planet.mbtiles"
```

## Common configuration
You can configure these values in `values.yaml` or via `--set` flags.

- Replicas
  - `replicaCount`: default `1`
- Image
  - `image.repository`: default `maptiler/tileserver-gl`
  - `image.tag`: defaults to chart `appVersion` if empty
  - `image.pullPolicy`: default `IfNotPresent`
- Service
  - `service.type`: default `ClusterIP`
  - `service.port`: default `8080`
- Ingress (disabled by default)
  - `ingress.enabled`: `true` to enable
  - `ingress.className`, `ingress.annotations`
  - `ingress.hosts[0].host`, `ingress.hosts[0].paths[*].{path,pathType}`
  - `ingress.tls` for TLS secrets and hosts
- Gateway API HTTPRoute (disabled by default)
  - `httpRoute.enabled`: `true` to enable
  - `httpRoute.parentRefs`, `httpRoute.hostnames`, `httpRoute.rules`
- Probes
  - `livenessProbe` / `readinessProbe` default to GET `/` on port named `http`
- Autoscaling (HPA)
  - `autoscaling.enabled`: `false` by default
  - `autoscaling.minReplicas`, `.maxReplicas`, `.targetCPUUtilizationPercentage`
- Scheduling
  - `nodeSelector`, `tolerations`, `affinity`
- Pod metadata
  - `podAnnotations`, `podLabels`
- ServiceAccount
  - `serviceAccount.create`, `serviceAccount.name`, `serviceAccount.annotations`, `serviceAccount.automount`
- Container arguments
  - `args`: passed directly to tileserver-gl
- Volumes
  - `volumes` and `volumeMounts`: add your data volumes here

## Examples

### A. Minimal working example (PVC with MBTiles, ClusterIP + port-forward)
Create a PVC named `my-tiles-pvc` that contains `zurich.mbtiles` at the root of the mounted volume, then use:

```
replicaCount: 1

volumes:
  - name: tiles
    persistentVolumeClaim:
      claimName: my-tiles-pvc

volumeMounts:
  - name: tiles
    mountPath: /data

args:
  - "--file"
  - "/data/zurich.mbtiles"
```

Install:
- `helm install my-tiles ./ -n maps`
- `kubectl -n maps port-forward svc/my-tiles-tileserver-gl-helm-chart 8080:8080`
- Open http://localhost:8080

### B. Expose via Ingress
```
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: tiles.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: tiles-tls
      hosts: [tiles.example.com]
```
Ensure your Ingress controller is installed and DNS/TLS are set up.

### C. Expose via Gateway API HTTPRoute
Requires Gateway API CRDs and a controller.
```
httpRoute:
  enabled: true
  parentRefs:
    - name: gateway
      sectionName: http
  hostnames:
    - tiles.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
```

## Upgrade, uninstall, and diff
- Upgrade with new values: `helm upgrade my-tiles ./ -n maps -f values.yaml`
- See pending changes: `helm diff upgrade my-tiles ./ -n maps -f values.yaml` (requires helm-diff plugin)
- Uninstall: `helm uninstall my-tiles -n maps`

## Tips and troubleshooting
- 404 or empty index: ensure your MBTiles path in `args` matches a file that exists inside the container
- Check logs: `kubectl -n <ns> logs deploy/<release>-tileserver-gl-helm-chart`
- Probe failures: verify the container serves on `/` and port `8080`, or adjust probes in values
- Ingress returns 404 from controller: verify host and path rules, and that Service name/port match

## Chart and app versions
- Chart version: see `Chart.yaml: version`
- App (image) version: defaults to `Chart.yaml: appVersion` if `image.tag` is empty
