---
layout: page
title: "OpenShift Cheat Sheet"
permalink: /cheatsheets/openshift/
---

_Red Hat OpenShift — alles was über Standard-Kubernetes hinausgeht._

## oc CLI Basics

```bash
# Login
oc login https://api.cluster.example.com:6443
oc login --token=sha256~xxxxx --server=https://api.cluster.example.com:6443
oc whoami                                    # aktueller User
oc whoami --show-server                      # API-Server URL
oc whoami --show-token                       # Token anzeigen

# Projekt (= Namespace mit Extras)
oc new-project my-project                    # erstellen & wechseln
oc project my-project                        # wechseln
oc projects                                  # alle anzeigen
oc status                                    # Projekt-Übersicht

# oc kann alles was kubectl kann!
oc get pods
oc describe deploy my-app
oc logs -f my-pod
oc exec -it my-pod -- /bin/sh
```

## Builds & Source-to-Image (S2I)

```bash
# App aus Git-Repository erstellen (S2I)
oc new-app https://github.com/user/repo.git
oc new-app python:3.11~https://github.com/user/repo.git   # Builder explizit
oc new-app --name=my-app --docker-image=nginx:latest       # aus Image

# Build starten / anzeigen
oc start-build my-app
oc start-build my-app --from-dir=.           # lokale Quellen hochladen
oc get builds
oc logs build/my-app-1                       # Build-Logs
oc cancel-build my-app-1
```

### BuildConfig

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-app
spec:
  source:
    type: Git
    git:
      uri: https://github.com/user/repo.git
      ref: main
  strategy:
    type: Source                              # S2I
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: java:17
        namespace: openshift
  output:
    to:
      kind: ImageStreamTag
      name: my-app:latest
  triggers:
    - type: GitHub
      github:
        secret: my-webhook-secret
    - type: ConfigChange
    - type: ImageChange
```

## ImageStreams

```bash
oc get is                                    # ImageStreams anzeigen
oc tag my-app:latest my-app:prod             # Tag setzen
oc import-image nginx --from=docker.io/nginx --confirm

# ImageStream zeigt auf externe Registry und trackt Änderungen
```

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: my-app
spec:
  lookupPolicy:
    local: true                              # Name statt voller Registry-URL
```

## DeploymentConfig vs. Deployment

```bash
# DeploymentConfig (OpenShift-spezifisch, legacy)
oc get dc
oc rollout latest dc/my-app
oc rollback dc/my-app

# Empfehlung: Standard-Kubernetes Deployment verwenden!
# DeploymentConfig ist deprecated seit OpenShift 4.14
```

## Routes (Ingress-Alternative)

```bash
oc expose svc/my-svc                         # HTTP-Route erstellen
oc expose svc/my-svc --hostname=app.example.com
oc get routes
```

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-route
spec:
  host: app.example.com
  to:
    kind: Service
    name: my-svc
  port:
    targetPort: 8080
  tls:
    termination: edge                        # edge | passthrough | reencrypt
    insecureEdgeTerminationPolicy: Redirect   # HTTP → HTTPS
```

## Security Context Constraints (SCC)

```bash
oc get scc                                   # alle SCCs anzeigen
oc describe scc restricted-v2                # Details
oc adm policy add-scc-to-user anyuid \
  -z my-sa -n my-project                     # SCC einem SA zuweisen
oc adm policy remove-scc-from-user anyuid \
  -z my-sa -n my-project                     # SCC entfernen

# Welche SCC wird verwendet?
oc get pod my-pod -o yaml | grep scc
```

```
Wichtige SCCs:
  restricted-v2    Default, kein Root, kein privileged
  nonroot-v2       Kein Root, aber mehr Capabilities
  anyuid           Beliebige UID (z.B. für nginx)
  privileged       Voller Zugriff (nur für Infra!)
```

## Templates & Catalog

```bash
# Template verarbeiten
oc process -f template.yaml \
  -p APP_NAME=my-app \
  -p IMAGE_TAG=1.0 | oc apply -f -

# Template-Parameter anzeigen
oc process --parameters -f template.yaml

# Katalog
oc get templates -n openshift                # verfügbare Templates
```

## User & Berechtigungen

```bash
# Rollen zuweisen
oc adm policy add-role-to-user admin alice -n my-project
oc adm policy add-role-to-user edit bob -n my-project
oc adm policy add-role-to-user view carol -n my-project
oc adm policy add-cluster-role-to-user cluster-admin admin-user

# Service Account
oc create sa my-sa
oc adm policy add-role-to-sa edit my-sa
oc sa get-token my-sa                        # Token abrufen (< 4.11)

# Wer kann was?
oc adm policy who-can get pods -n my-project
oc auth can-i create deployments             # eigene Rechte prüfen
```

## Wichtige OpenShift-Ressourcen

```
Ressource                  Kurz      Beschreibung
────────────────────────── ───────── ─────────────────────────
Route                      route     Externe URL → Service
BuildConfig                bc        Build-Definition
Build                      build     Build-Instanz
ImageStream                is        Image-Referenz mit Tags
ImageStreamTag             istag     Einzelner Tag
DeploymentConfig (legacy)  dc        OpenShift Deployment
Project                    project   Namespace + Extras
SecurityContextConstraints scc       Pod-Security-Policies
Template                   template  Parametrisierte Manifeste
```

## oc vs kubectl

```
oc exklusiv                        Beide (oc = kubectl + mehr)
────────────────────────────────── ──────────────────────────
oc new-app                         get, describe, logs, exec
oc new-project                     apply, delete, edit, patch
oc start-build                     scale, rollout
oc expose (Route)                  port-forward, cp
oc adm (Cluster-Admin)             top, auth, api-resources
oc login (OAuth)                   config (kubeconfig)
```
