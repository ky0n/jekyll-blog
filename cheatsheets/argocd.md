---
layout: page
title: "Argo CD Cheat Sheet"
permalink: /cheatsheets/argocd/
---

_GitOps Continuous Delivery f&uuml;r Kubernetes._

## Konzepte

```
Application     → beschreibt SOLL-Zustand (Git-Repo → Cluster)
AppProject      → Gruppierung + Berechtigungen für Applications
Sync            → Git-Zustand auf Cluster anwenden
Health          → Status der deployen Ressourcen
Refresh         → Git-Repo auf Änderungen prüfen
```

## argocd CLI

```bash
# Login
argocd login argocd.example.com              # interaktiv
argocd login argocd.example.com \
  --username admin --password secret          # non-interaktiv
argocd login argocd.example.com \
  --sso                                       # SSO/OAuth

# Kontext
argocd context                               # aktueller Server
argocd account get-user-info                 # eingeloggter User
```

## Application verwalten

```bash
# Erstellen
argocd app create my-app \
  --repo https://github.com/user/repo.git \
  --path k8s/overlays/prod \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace my-ns \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Anzeigen
argocd app list
argocd app get my-app                        # Details + Sync-Status
argocd app get my-app --refresh              # Git refreshen + Status

# Sync
argocd app sync my-app                       # manueller Sync
argocd app sync my-app --prune               # gelöschte Ressourcen entfernen
argocd app sync my-app --force               # Force-Sync
argocd app sync my-app \
  --resource apps/Deployment/my-deploy       # einzelne Ressource syncen

# Diff
argocd app diff my-app                       # Unterschiede Git ↔ Cluster

# Rollback
argocd app rollback my-app <revision>
argocd app history my-app                    # Sync-Historie

# Löschen
argocd app delete my-app                     # App + Ressourcen
argocd app delete my-app --cascade=false     # nur App, Ressourcen bleiben
```

## Application YAML

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd                          # ArgoCD Namespace
  finalizers:
    - resources-finalizer.argocd.argoproj.io  # Ressourcen bei App-Delete löschen
spec:
  project: default

  source:
    repoURL: https://github.com/user/repo.git
    targetRevision: main                     # Branch, Tag oder HEAD
    path: k8s/overlays/prod                  # Pfad zu Manifests

    # Kustomize
    kustomize:
      images:
        - my-app=registry.io/my-app:v1.2.3

    # Helm
    # helm:
    #   releaseName: my-app
    #   valueFiles:
    #     - values-prod.yaml
    #   values: |
    #     replicas: 3
    #   parameters:
    #     - name: image.tag
    #       value: "v1.2.3"

  destination:
    server: https://kubernetes.default.svc   # In-Cluster
    namespace: my-namespace

  syncPolicy:
    automated:
      prune: true                            # gelöschte Ressourcen entfernen
      selfHeal: true                         # manuelle Drift korrigieren
      allowEmpty: false                      # Schutz vor leerem Repo
    syncOptions:
      - CreateNamespace=true                 # NS automatisch erstellen
      - PruneLast=true                       # erst deployen, dann prunen
      - ApplyOutOfSyncOnly=true              # nur geänderte Ressourcen
      - ServerSideApply=true                 # Server-Side Apply
    retry:
      limit: 3
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 1m

  ignoreDifferences:                         # Felder ignorieren
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas                     # z.B. wenn HPA skaliert
```

## ApplicationSet

Generiert mehrere Applications aus einer Vorlage.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-apps
  namespace: argocd
spec:
  generators:
    # Liste
    - list:
        elements:
          - cluster: dev
            url: https://dev-cluster.example.com
            namespace: my-app-dev
          - cluster: prod
            url: https://prod-cluster.example.com
            namespace: my-app-prod

    # Git-Verzeichnisse (ein App pro Verzeichnis)
    # - git:
    #     repoURL: https://github.com/user/repo.git
    #     revision: main
    #     directories:
    #       - path: apps/*

    # Cluster Generator (alle registrierten Cluster)
    # - clusters:
    #     selector:
    #       matchLabels:
    #         env: production

  template:
    metadata:
      name: 'my-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/user/repo.git
        targetRevision: main
        path: 'k8s/overlays/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: my-team
  namespace: argocd
spec:
  description: "Projekt für Team A"
  sourceRepos:
    - https://github.com/my-org/*            # erlaubte Repos
  destinations:
    - server: https://kubernetes.default.svc
      namespace: 'team-a-*'                  # erlaubte Namespaces
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
  roles:
    - name: developer
      policies:
        - p, proj:my-team:developer, applications, sync, my-team/*, allow
        - p, proj:my-team:developer, applications, get, my-team/*, allow
      groups:
        - my-team-devs                       # OIDC Group
```

## Sync Waves & Hooks

```yaml
# Sync-Reihenfolge steuern (niedrigere Welle zuerst)
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"       # vor Standard (0)

# Typische Reihenfolge:
#  -2  Namespace, RBAC
#  -1  ConfigMaps, Secrets
#   0  Deployments, Services (default)
#   1  Ingress, Routes
#   2  Post-Deploy Jobs

---
# Sync Hooks (Jobs die bei bestimmten Events laufen)
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync         # vor dem Sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded

# Hook-Typen:
#   PreSync    → vor dem Sync (z.B. DB-Migration)
#   Sync       → während des Syncs
#   PostSync   → nach erfolgreichem Sync (z.B. Tests, Notification)
#   SyncFail   → bei fehlgeschlagenem Sync
```

## Notifications

```yaml
# ConfigMap für Notification Templates
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [slack-message]
  template.slack-message: |
    message: |
      App {{.app.metadata.name}} synced successfully!
      Revision: {{.app.status.sync.revision}}
  service.slack: |
    token: $slack-token
```

## Status & Troubleshooting

```bash
# App-Status
argocd app get my-app
# Sync Status:  Synced / OutOfSync
# Health:       Healthy / Degraded / Progressing / Missing

# Detaillierte Ressourcen
argocd app resources my-app
argocd app manifests my-app                  # gerenderte Manifests

# Logs der managed Pods
argocd app logs my-app

# ArgoCD Server Logs
kubectl logs -n argocd deploy/argocd-server
kubectl logs -n argocd deploy/argocd-repo-server
kubectl logs -n argocd deploy/argocd-application-controller

# Hard Refresh (Cache leeren)
argocd app get my-app --hard-refresh
```

## Quick Reference

```
Aktion                    CLI                              UI
───────────────────────── ──────────────────────────────── ────────
App erstellen             argocd app create ...            + New App
Sync auslösen             argocd app sync my-app           Sync Button
Diff anzeigen             argocd app diff my-app           App Detail
Rollback                  argocd app rollback my-app REV   History Tab
Status prüfen             argocd app get my-app            Dashboard
Repo hinzufügen           argocd repo add URL              Settings
Cluster hinzufügen        argocd cluster add CONTEXT       Settings
```
