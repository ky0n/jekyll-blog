---
layout: page
title: "Tekton Cheat Sheet"
permalink: /cheatsheets/tekton/
---

_Cloud-native CI/CD Pipelines auf Kubernetes._

## Konzepte

```
Step        → ein Container, der einen Befehl ausführt
Task        → eine Reihe von Steps (läuft in einem Pod)
Pipeline    → eine Reihe von Tasks (DAG)
TaskRun     → Ausführung eines Tasks
PipelineRun → Ausführung einer Pipeline
Workspace   → gemeinsamer Speicher zwischen Steps/Tasks
```

## tkn CLI

```bash
# Tasks
tkn task list
tkn task describe my-task
tkn task start my-task                       # TaskRun starten
tkn task delete my-task

# TaskRuns
tkn taskrun list
tkn taskrun describe my-taskrun
tkn taskrun logs my-taskrun                  # Logs anzeigen
tkn taskrun logs -f my-taskrun               # follow
tkn taskrun delete my-taskrun

# Pipelines
tkn pipeline list
tkn pipeline describe my-pipeline
tkn pipeline start my-pipeline               # PipelineRun starten
tkn pipeline start my-pipeline \
  -p git-url=https://github.com/user/repo \
  -p image=my-registry/my-app:latest \
  -w name=shared-workspace,claimName=my-pvc

# PipelineRuns
tkn pipelinerun list
tkn pipelinerun describe my-pipelinerun
tkn pipelinerun logs my-pipelinerun
tkn pipelinerun logs -f --last               # letzter Run, follow
tkn pipelinerun cancel my-pipelinerun
tkn pipelinerun delete my-pipelinerun

# Hub (vorgefertigte Tasks)
tkn hub search git-clone
tkn hub install task git-clone
```

## Task

```yaml
apiVersion: tekton.dev/v1
kind: Task
metadata:
  name: build-and-test
spec:
  params:
    - name: git-url
      type: string
    - name: verbose
      type: string
      default: "false"

  workspaces:
    - name: source

  results:
    - name: commit-sha
      description: "Git commit SHA"

  steps:
    - name: clone
      image: alpine/git:latest
      script: |
        git clone $(params.git-url) $(workspaces.source.path)/repo
        cd $(workspaces.source.path)/repo
        echo -n $(git rev-parse HEAD) > $(results.commit-sha.path)

    - name: test
      image: maven:3.9-eclipse-temurin-21
      workingDir: $(workspaces.source.path)/repo
      script: |
        mvn test

    - name: build
      image: maven:3.9-eclipse-temurin-21
      workingDir: $(workspaces.source.path)/repo
      script: |
        mvn package -DskipTests
```

## Pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-deploy
spec:
  params:
    - name: git-url
      type: string
    - name: image
      type: string

  workspaces:
    - name: shared-workspace

  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone              # aus Tekton Hub
      params:
        - name: url
          value: $(params.git-url)
      workspaces:
        - name: output
          workspace: shared-workspace

    - name: build
      taskRef:
        name: build-and-test
      runAfter:
        - fetch-source               # Abhängigkeit
      params:
        - name: git-url
          value: $(params.git-url)
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: build-image
      taskRef:
        name: buildah
      runAfter:
        - build
      params:
        - name: IMAGE
          value: $(params.image)
      workspaces:
        - name: source
          workspace: shared-workspace

    - name: deploy
      taskRef:
        name: deploy-to-k8s
      runAfter:
        - build-image
      params:
        - name: image
          value: $(params.image)

  finally:                           # läuft immer (auch bei Fehler)
    - name: notify
      taskRef:
        name: send-notification
      params:
        - name: status
          value: $(tasks.status)     # Succeeded | Failed
```

## PipelineRun

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: build-deploy-run-
spec:
  pipelineRef:
    name: build-deploy
  params:
    - name: git-url
      value: https://github.com/user/repo.git
    - name: image
      value: my-registry.io/my-app:latest
  workspaces:
    - name: shared-workspace
      persistentVolumeClaim:
        claimName: pipeline-pvc
  timeouts:
    pipeline: "1h"
    tasks: "30m"
```

## Workspaces

```yaml
# In PipelineRun definieren:
workspaces:
  # PVC (persistent)
  - name: shared-data
    persistentVolumeClaim:
      claimName: my-pvc

  # EmptyDir (temporär, nur für Pipeline-Laufzeit)
  - name: temp
    emptyDir: {}

  # ConfigMap (read-only)
  - name: config
    configMap:
      name: my-config

  # Secret
  - name: credentials
    secret:
      secretName: my-secret

  # VolumeClaimTemplate (automatisch PVC pro Run)
  - name: source
    volumeClaimTemplate:
      spec:
        accessModes: [ReadWriteOnce]
        resources:
          requests:
            storage: 1Gi
```

## Trigger (Event-basierte Pipelines)

```yaml
# EventListener — empfängt Webhooks
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: tekton-triggers-sa
  triggers:
    - name: github-push
      interceptors:
        - ref:
            name: github
          params:
            - name: secretRef
              value:
                secretName: github-webhook-secret
                secretKey: token
            - name: eventTypes
              value: ["push"]
      bindings:
        - ref: github-binding
      template:
        ref: pipeline-template

---
# TriggerBinding — extrahiert Daten aus Event
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: github-binding
spec:
  params:
    - name: git-url
      value: $(body.repository.clone_url)
    - name: git-revision
      value: $(body.after)

---
# TriggerTemplate — erstellt PipelineRun
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-template
spec:
  params:
    - name: git-url
    - name: git-revision
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: triggered-run-
      spec:
        pipelineRef:
          name: build-deploy
        params:
          - name: git-url
            value: $(tt.params.git-url)
```

## Nützliche Patterns

```yaml
# When-Bedingung (Task nur bei Bedingung ausführen)
tasks:
  - name: deploy-prod
    when:
      - input: $(params.branch)
        operator: in
        values: ["main"]
    taskRef:
      name: deploy

# Task-Ergebnis an nächsten Task übergeben
tasks:
  - name: build
    taskRef:
      name: build-task
  - name: deploy
    params:
      - name: image-digest
        value: $(tasks.build.results.image-digest)

# Matrix (parallele Ausführung mit verschiedenen Params)
tasks:
  - name: test
    matrix:
      params:
        - name: java-version
          value: ["17", "21", "25"]
    taskRef:
      name: run-tests
```

## Quick Reference

```
Ressource         tkn Befehl                Kubernetes Befehl
───────────────── ─────────────────────────  ──────────────────
Task              tkn task list             kubectl get task
TaskRun           tkn taskrun list          kubectl get taskrun
Pipeline          tkn pipeline list         kubectl get pipeline
PipelineRun       tkn pipelinerun list      kubectl get pipelinerun
EventListener     —                         kubectl get el
TriggerBinding    —                         kubectl get tb
TriggerTemplate   —                         kubectl get tt
```
