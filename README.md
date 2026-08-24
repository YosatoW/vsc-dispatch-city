# Dispatch City – Block 03 und 04

Dieses Repository dokumentiert den Aufbau von **Dispatch City ab Block 03**. Der aktuelle Projektstand umfasst:

- Block 03: Deployments, Services und ConfigMaps
- Block 04: Ingress und externe Zugriffe

## Releases

- `v1.0.0`: Block 03 – Foundation mit Deployments, Services und ConfigMaps
- `v1.1.1`: Block 04 – Ingress, Traefik, zwei Dashboard-Replikas und Self-Healing

## Voraussetzungen

- Docker Desktop
- Git
- k3d
- kubectl
- PowerShell unter Windows oder Bash, WSL beziehungsweise Git Bash
- Browser

Versionen prüfen:

```powershell
docker version
k3d version
kubectl version --client
git --version
```

## Repository klonen

```powershell
git clone https://github.com/YosatoW/vsc-dispatch-city.git
cd vsc-dispatch-city
```

## Cluster erstellen oder starten

Vorhandene Cluster anzeigen:

```powershell
k3d cluster list
```

Falls `teko-k8s` fehlt:

```powershell
k3d cluster create teko-k8s --agents 2 --wait
```

Falls der Cluster gestoppt ist:

```powershell
k3d cluster start teko-k8s
```

Kontext setzen und Nodes prüfen:

```powershell
kubectl config use-context k3d-teko-k8s
kubectl get nodes -o wide
```

Erwartet werden ein Server-Node und zwei Agent-Nodes im Status `Ready`.

# Block 03 – Deployments, Services und ConfigMaps

## Was umgesetzt wurde

- Dispatch-City-Foundation `v1.0.0` übernommen
- Images für Control API und Dashboard gebaut
- Images in den k3d-Cluster importiert
- Kubernetes-Manifeste mit Kustomize gerendert und angewendet
- Namespace `food-delivery` verwendet
- Control API und Dashboard als Deployments betrieben
- Services, EndpointSlices und Kubernetes-DNS geprüft
- ConfigMap `simulation-config` verwendet
- Readiness- und Liveness-Probes geprüft
- Änderungen an `TICK_MS` durch einen Rollout übernommen

## Images bauen

```powershell
docker build -t food-delivery-control-api:local --build-arg SERVICE=control-api -f build/go-service.Dockerfile .
docker build -t food-delivery-dashboard:local ./apps/dashboard
```

## Images in k3d importieren

```powershell
k3d image import -c teko-k8s food-delivery-control-api:local food-delivery-dashboard:local
```

## Geplanten Stand anzeigen

```powershell
kubectl kustomize ./deploy/overlays/block-03-standalone
```

Der Befehl rendert die Manifeste, verändert den Cluster aber noch nicht.

## Block 03 deployen

```powershell
kubectl apply -k ./deploy/overlays/block-03-standalone
kubectl -n food-delivery rollout status deployment/control-api
kubectl -n food-delivery rollout status deployment/dashboard
kubectl -n food-delivery get deploy,pods,svc,cm -o wide
```

## Service, EndpointSlice und DNS prüfen

```powershell
kubectl -n food-delivery get svc,endpointslice
kubectl -n food-delivery get pods --show-labels
kubectl -n food-delivery get endpointslice -l kubernetes.io/service-name=control-api
```

Health-Endpunkt aus einem temporären Pod prüfen:

```powershell
kubectl run dns-test --image=curlimages/curl:8.8.0 -n food-delivery --restart=Never --rm -i -- curl -fsS http://control-api:8080/health/ready
```

Vollständiger interner DNS-Name:

```text
http://control-api.food-delivery.svc.cluster.local:8080/health/ready
```

## ConfigMap und Rollout

```powershell
kubectl diff -k ./deploy/overlays/block-03-standalone
kubectl apply -k ./deploy/overlays/block-03-standalone
kubectl -n food-delivery rollout restart deployment/control-api
kubectl -n food-delivery rollout status deployment/control-api
kubectl -n food-delivery logs deployment/control-api
```

# Block 04 – Ingress und externe Zugriffe

## Was umgesetzt wurde

- Block-4-Erweiterung `v1.1.1` integriert
- Ingress `food-delivery` mit Ingress-Klasse `traefik` erstellt
- Dashboard auf zwei Replikas skaliert
- gemeinsames Path Routing eingerichtet
- Dashboard, API und Health-Endpunkt über denselben Einstiegspunkt getestet
- Load Balancing zwischen zwei Dashboard-Pods sichtbar gemacht
- Self-Healing durch Löschen eines Dashboard-Pods geprüft

## Architektur

```text
Browser / Client
       |
       | HTTP, zum Beispiel localhost:8080
       v
Traefik Ingress Controller
       |
       v
Ingress food-delivery
       |
       +-- /         -> dashboard:3000 -> Dashboard-Pod A oder B
       +-- /api      -> control-api:8080
       +-- /health   -> control-api:8080
       +-- /metrics  -> control-api:8080
```

## Block-04-Erweiterung installieren

Nur erforderlich, wenn der Block-04-Stand noch nicht im Repository enthalten ist:

```powershell
git clone --branch v1.1.1 --depth 1 https://github.com/SwitzerChees/vsc-dispatch-city-04-ingress.git ..\vsc-dispatch-city-04-ingress
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
& "..\vsc-dispatch-city-04-ingress\install.ps1" -Target "."
```

## Geplanten Block-04-Stand anzeigen

```powershell
kubectl kustomize ./deploy/overlays/block-04-ingress
```

Erwartete Kerneinstellungen:

- Ingress: `food-delivery`
- Ingress-Klasse: `traefik`
- Dashboard-Replikas: `2`
- Control-API-Replikas: `1`

## Images bauen und importieren

```powershell
docker build -t food-delivery-control-api:local --build-arg SERVICE=control-api -f build/go-service.Dockerfile .
docker build -t food-delivery-dashboard:local ./apps/dashboard
k3d image import -c teko-k8s food-delivery-control-api:local food-delivery-dashboard:local
```

## Block 04 deployen

```powershell
kubectl apply -k ./deploy/overlays/block-04-ingress
kubectl -n food-delivery rollout status deployment/control-api --timeout=180s
kubectl -n food-delivery rollout status deployment/dashboard --timeout=180s
kubectl -n food-delivery get deployment,ingress
```

Erwarteter Zustand:

```text
control-api   1/1
dashboard     2/2
Ingress       traefik
```

## Anwendung öffnen

In einem separaten Terminal:

```powershell
kubectl -n kube-system rollout status deployment/traefik --timeout=180s
kubectl -n kube-system port-forward service/traefik 8080:80
```

Das Terminal muss geöffnet bleiben. Falls Port `8080` belegt ist:

```powershell
kubectl -n kube-system port-forward service/traefik 8081:80
```

Browser:

```text
http://127.0.0.1:8080/
```

Alternativ:

```text
http://127.0.0.1:8081/
```

## Routen testen

```powershell
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8080/").StatusCode
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8080/api/v1/snapshot").StatusCode
(Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:8080/health/ready").StatusCode
```

Erwartet:

```text
200
200
200
```

## Load Balancing prüfen

```powershell
1..20 | ForEach-Object {
    (Invoke-RestMethod -Uri "http://127.0.0.1:8080/ui-instance" -DisableKeepAlive).instance
} | Sort-Object -Unique
```

EndpointSlice prüfen:

```powershell
kubectl -n food-delivery get endpointslice -l kubernetes.io/service-name=dashboard -o wide
```

## Self-Healing prüfen

Pods beobachten:

```powershell
kubectl -n food-delivery get pods -l app.kubernetes.io/name=dashboard -w
```

In einem zweiten Terminal einen Pod löschen:

```powershell
$pod = kubectl -n food-delivery get pod -l app.kubernetes.io/name=dashboard -o jsonpath='{.items[0].metadata.name}'
kubectl -n food-delivery delete pod $pod
kubectl -n food-delivery rollout status deployment/dashboard --timeout=180s
```

Kubernetes erstellt automatisch einen Ersatz-Pod und stellt wieder zwei Dashboard-Replikas bereit.

# Diagnose

```powershell
k3d cluster list
kubectl config current-context
kubectl get nodes -o wide
kubectl -n food-delivery get deploy,pods,svc,endpointslice,ingress,cm -o wide
kubectl -n food-delivery get events --sort-by='.lastTimestamp'
kubectl -n food-delivery describe ingress food-delivery
kubectl -n food-delivery logs deployment/control-api
```

## Häufige Fehler

### Port bereits belegt

Einen anderen lokalen Port verwenden, zum Beispiel `8081:80`.

### ImagePullBackOff

Images erneut bauen und in `teko-k8s` importieren.

### Webseite nicht erreichbar

- Docker Desktop prüfen
- Cluster `teko-k8s` prüfen
- Kontext `k3d-teko-k8s` setzen
- Pods und Rollouts prüfen
- Traefik-Port-Forward neu starten

### 404 über Traefik

```powershell
kubectl -n food-delivery describe ingress food-delivery
```

# Git-Workflow

Vor Arbeitsbeginn:

```powershell
git pull
```

Änderungen hochladen:

```powershell
git status
git add .
git commit -m "Describe change"
git push
```

## Release-Tags erstellen

Block 03 markieren:

```powershell
git tag -a v1.0.0 -m "Block 03 foundation release v1.0.0"
git push origin v1.0.0
```

Block 04 markieren:

```powershell
git tag -a v1.1.1 -m "Block 04 ingress release v1.1.1"
git push origin v1.1.1
```

# Aufräumen

Nur Dispatch City entfernen:

```powershell
kubectl delete namespace food-delivery
```

Vollständiger Reset:

```powershell
k3d cluster delete teko-k8s
```
