---
layout: page
title: "Kubernetes Cheat Sheet"
permalink: /cheatsheets/kubernetes/
---

_Kompakte Referenz für den Alltag mit kubectl & Kubernetes._

## kubectl Basics

```bash
# Kontext & Cluster
kubectl config get-contexts                  # alle Kontexte anzeigen
kubectl config use-context my-cluster        # Kontext wechseln
kubectl config current-context               # aktuellen Kontext anzeigen
kubectl cluster-info                         # Cluster-Info

# Namespace
kubectl get ns                               # alle Namespaces
kubectl config set-context --current --namespace=my-ns   # Default-NS setzen
```

## Ressourcen anzeigen

```bash
kubectl get pods                             # Pods im aktuellen NS
kubectl get pods -A                          # alle Namespaces
kubectl get pods -o wide                     # mehr Details (Node, IP)
kubectl get pods -o yaml                     # vollständiges YAML
kubectl get pods -w                          # watch (live Updates)
kubectl get pods --show-labels               # Labels anzeigen
kubectl get pods -l app=nginx                # nach Label filtern
kubectl get pods --field-selector status.phase=Running

kubectl get all                              # Pods, Services, Deployments, etc.
kubectl get deploy,svc,ing                   # mehrere Typen
kubectl api-resources                        # alle verfügbaren Ressource-Typen
```

## Ressourcen beschreiben & debuggen

```bash
kubectl describe pod my-pod                  # Details + Events
kubectl logs my-pod                          # Logs
kubectl logs my-pod -c my-container          # Container im Multi-Container Pod
kubectl logs my-pod --previous               # Logs des letzten Crashs
kubectl logs -f my-pod                       # follow (tail -f)
kubectl logs -l app=nginx --all-containers   # Logs aller Pods mit Label

kubectl exec -it my-pod -- /bin/sh           # Shell im Pod
kubectl exec my-pod -- cat /etc/config       # Einzelnen Befehl ausführen
kubectl port-forward my-pod 8080:80          # Port weiterleiten
kubectl port-forward svc/my-svc 8080:80      # Service Port weiterleiten

kubectl top pods                             # CPU/Memory pro Pod
kubectl top nodes                            # CPU/Memory pro Node
```

## Erstellen & Anwenden

```bash
# Deklarativ (empfohlen)
kubectl apply -f manifest.yaml               # erstellen oder updaten
kubectl apply -f ./manifests/                 # ganzes Verzeichnis
kubectl apply -k ./overlays/prod/            # mit Kustomize

# Imperativ (schnell, zum Testen)
kubectl run nginx --image=nginx              # Pod erstellen
kubectl create deploy nginx --image=nginx    # Deployment erstellen
kubectl expose deploy nginx --port=80        # Service erstellen

# Dry-Run: YAML generieren ohne anzuwenden
kubectl create deploy nginx --image=nginx \
  --dry-run=client -o yaml > deploy.yaml

kubectl run tmp --image=busybox --rm -it \
  --restart=Never -- wget -qO- http://my-svc  # temporärer Debug-Pod
```

## Bearbeiten & Löschen

```bash
kubectl edit deploy my-deploy                # live editieren
kubectl patch deploy my-deploy \
  -p '{"spec":{"replicas":3}}'               # einzelnes Feld patchen
kubectl set image deploy/my-deploy \
  app=nginx:1.25                             # Image updaten

kubectl delete pod my-pod                    # Pod löschen
kubectl delete -f manifest.yaml              # aus Manifest löschen
kubectl delete pods -l app=test              # nach Label löschen
kubectl delete pod my-pod --grace-period=0 \
  --force                                    # sofort löschen
```

## Skalieren & Rollouts

```bash
kubectl scale deploy my-deploy --replicas=5
kubectl autoscale deploy my-deploy \
  --min=2 --max=10 --cpu-percent=80          # HPA erstellen

# Rollout
kubectl rollout status deploy/my-deploy      # Status
kubectl rollout history deploy/my-deploy     # Versionen
kubectl rollout undo deploy/my-deploy        # Rollback
kubectl rollout undo deploy/my-deploy \
  --to-revision=2                            # zu bestimmter Version
kubectl rollout restart deploy/my-deploy     # Neustart aller Pods
kubectl rollout pause deploy/my-deploy       # pausieren
kubectl rollout resume deploy/my-deploy      # fortsetzen
```

## Pod Spec — wichtige Felder

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: my-app
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
      env:
        - name: DB_HOST
          value: "postgres"
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "500m"
          memory: "256Mi"
      livenessProbe:
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
      readinessProbe:
        httpGet:
          path: /ready
          port: 80
      volumeMounts:
        - name: config
          mountPath: /etc/config
  volumes:
    - name: config
      configMap:
        name: my-config
  restartPolicy: Always
  serviceAccountName: my-sa
```

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: my-app:1.0
          ports:
            - containerPort: 8080
```

## Service

```yaml
# ClusterIP (intern)
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP          # default

# NodePort (extern via Node-Port)
  type: NodePort
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080      # 30000-32767

# LoadBalancer (Cloud)
  type: LoadBalancer

# Headless (für StatefulSets, DNS pro Pod)
  clusterIP: None
```

## Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

## ConfigMap & Secret

```bash
# Erstellen
kubectl create configmap my-config \
  --from-literal=key1=value1 \
  --from-file=config.properties

kubectl create secret generic my-secret \
  --from-literal=password=s3cret \
  --from-file=tls.crt
```

```yaml
# ConfigMap als Env oder Volume
env:
  - name: MY_KEY
    valueFrom:
      configMapKeyRef:
        name: my-config
        key: key1
envFrom:
  - configMapRef:
      name: my-config          # alle Keys als Env-Vars
volumes:
  - name: config
    configMap:
      name: my-config          # als Dateien gemountet
```

## Jobs & CronJobs

```yaml
# Job — einmaliger Task
apiVersion: batch/v1
kind: Job
metadata:
  name: my-job
spec:
  backoffLimit: 3
  template:
    spec:
      containers:
        - name: job
          image: busybox
          command: ["echo", "done"]
      restartPolicy: Never

---
# CronJob — wiederkehrend
apiVersion: batch/v1
kind: CronJob
metadata:
  name: my-cronjob
spec:
  schedule: "*/5 * * * *"       # alle 5 Minuten
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cron
              image: busybox
              command: ["echo", "tick"]
          restartPolicy: OnFailure
```

## RBAC

```yaml
# Role (Namespace-spezifisch)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "watch", "list"]

---
# RoleBinding
kind: RoleBinding
metadata:
  name: read-pods
subjects:
  - kind: ServiceAccount
    name: my-sa
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

# ClusterRole / ClusterRoleBinding → cluster-weit
```

## Quick Reference

```
Ressource        Kurz    Erstellen
──────────────── ─────── ──────────────────────────
Pod              po      kubectl run
Deployment       deploy  kubectl create deploy
Service          svc     kubectl expose
ConfigMap        cm      kubectl create configmap
Secret           secret  kubectl create secret
Ingress          ing     kubectl create ingress
Job              job     kubectl create job
CronJob          cj      kubectl create cronjob
Namespace        ns      kubectl create ns
ServiceAccount   sa      kubectl create sa
PersistentVolumeClaim pvc kubectl apply -f
Node             no      —
```
