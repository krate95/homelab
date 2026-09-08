# Traefik en Talos

Guía de implementación de Traefik como Ingress Controller, adaptada a partir de
la configuración existente en `~/homelab` (donde Traefik viene integrado con
K3s vía `HelmChartConfig`). Talos no trae Traefik empaquetado, así que aquí se
instala como un `HelmRelease` de Flux propio, siguiendo el mismo patrón que
`infrastructure/metallb`.

## Diferencias frente a `~/homelab`

- **homelab (K3s)**: Traefik ya viene instalado por K3s; solo se configura
  declarativamente con `HelmChartConfig` (`helm.cattle.io/v1`, un CRD
  específico de K3s).
- **taloslab (Talos)**: Traefik no existe por defecto. Hay que instalarlo con
  su propio `HelmRepository` + `HelmRelease`.
- **CIDR de pods**: homelab permite `10.42.0.0/16` (rango de K3s) en el
  middleware `local-only`. Este clúster usa Flannel con `10.244.0.0/16`
  (verificado con `kubectl get pods -A -o wide`), así que el middleware debe
  usar ese rango.
- **IngressClass**: el chart oficial de Traefik ya crea el `IngressClass`
  automáticamente (`ingressClass.enabled: true` por defecto), así que no hace
  falta el archivo manual `ingressclass.yaml` que sí existe en homelab.

## Por qué se divide en dos Kustomizations

El `HelmRelease` de Traefik instala sus propios CRDs (`Middleware`,
`IngressRoute`, etc.) junto con el chart. Si el `Middleware` `local-only` se
aplicara en el mismo lote (`kustomize build`) que el `HelmRelease`, Flux
fallaría con:

```
no matches for kind "Middleware" in version "traefik.io/v1alpha1"
```

porque en el momento del `dry-run` los CRDs aún no existen (el `HelmRelease`
se instala de forma asíncrona por `helm-controller`). Es el mismo problema que
ya resolvimos con `metallb` / `metallb-config`.

Por eso:
- `infrastructure/traefik/` instala solo el chart (namespace, HelmRepository,
  HelmRelease) y forma parte de la Kustomization `infrastructure`.
- `infrastructure/traefik-config/` aplica el `Middleware`, en una Kustomization
  de Flux **separada** con `dependsOn: infrastructure`, para garantizar que
  los CRDs ya existen.

`infrastructure/longhorn/middlewares.yaml` tiene el mismo problema (usa
`Middleware` de Traefik) y debe moverse a `infrastructure/longhorn-config/`
con el mismo patrón de dependencia.

## Fuentes usadas

- [`values.yaml` oficial del chart](https://raw.githubusercontent.com/traefik/traefik-helm-chart/master/traefik/values.yaml) —
  confirma `ingressClass.enabled: true` e `ingressClass.isDefaultClass: true`
  por defecto, y la estructura de `deployment.kind` / `resources`.
- [ArtifactHub - traefik/traefik](https://artifacthub.io/packages/helm/traefik/traefik) —
  versión estable actual del chart: `41.5.0`.
- [Flux docs - Kustomization dependency management](https://fluxcd.io/flux/components/kustomize/kustomizations/#dependency-management) —
  patrón oficial de `dependsOn` + `wait: true` para CRDs instalados por Helm.
- [Traefik - Getting Started on Kubernetes](https://doc.traefik.io/traefik/getting-started/kubernetes/#install-traefik) —
  confirma la URL del repo Helm (`https://traefik.github.io/charts`) y que el
  provider `kubernetesCRD` está habilitado por defecto.
- `~/homelab/clusters/roshar/infrastructure/traefik/` — configuración real de
  referencia (namespace, middleware `local-only`, valores de recursos).
- `kubectl get pods -A -o wide` contra este clúster — para confirmar el CIDR
  de pods real (Flannel, `10.244.0.0/16`) en vez de asumir el de homelab.

## Pasos

### 1. `infrastructure/traefik/namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik
```

*(ya existe)*

### 2. `infrastructure/traefik/helmrepository.yaml`

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: traefik
  namespace: flux-system
spec:
  interval: 1h
  url: https://traefik.github.io/charts
```

### 3. `infrastructure/traefik/helmrelease.yaml`

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: flux-system
spec:
  interval: 30m
  targetNamespace: traefik
  releaseName: traefik
  chart:
    spec:
      chart: traefik
      version: "41.5.0"
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
  install:
    createNamespace: false
  upgrade:
    remediation:
      retries: 3
  values:
    deployment:
      kind: DaemonSet
    priorityClassName: homelab-critical
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        memory: 512Mi
    service:
      spec:
        externalTrafficPolicy: Local
```

### 4. `infrastructure/traefik/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
```

### 5. Añadir `traefik` a `infrastructure/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - priority-classes.yaml
  - storage
  - longhorn
  - metallb
  - traefik
```

### 6. `infrastructure/traefik-config/middleware-local-only.yaml`

```yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: local-only
  namespace: traefik
spec:
  ipAllowList:
    sourceRange:
      - 192.168.1.0/24
      - 10.244.0.0/16
```

### 7. `infrastructure/traefik-config/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - middleware-local-only.yaml
```

### 8. `clusters/roshar/traefik-config.yaml`

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: traefik-config
  namespace: flux-system
spec:
  interval: 1h
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/traefik-config
  prune: true
  wait: true
  timeout: 5m
  dependsOn:
    - name: infrastructure
```

### 9. Mover `longhorn/middlewares.yaml` (mismo problema)

```bash
mkdir -p infrastructure/longhorn-config
git mv infrastructure/longhorn/middlewares.yaml infrastructure/longhorn-config/middlewares.yaml
```

Quitar `middlewares.yaml` de `infrastructure/longhorn/kustomization.yaml`.

`infrastructure/longhorn-config/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - middlewares.yaml
```

`clusters/roshar/longhorn-config.yaml` (igual que `traefik-config.yaml`, cambiando `name` a `longhorn-config` y `path` a `./infrastructure/longhorn-config`).

### 10. Commit y push

```bash
git add infrastructure/traefik infrastructure/traefik-config infrastructure/longhorn-config \
        infrastructure/longhorn/kustomization.yaml infrastructure/kustomization.yaml \
        clusters/roshar/traefik-config.yaml clusters/roshar/longhorn-config.yaml
git commit -m "add traefik HelmRelease and split CRD-dependent middlewares"
git push
```

## Verificación

```bash
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source
flux get kustomizations -A
```

`infrastructure`, `traefik-config` y `longhorn-config` deben quedar `READY: True`.
