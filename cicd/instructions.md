# Helm Templating: Complete Tutorial & Reference Guide

Welcome to the definitive tutorial on **Helm Templating**. This guide covers Helm's templating engine from basic syntax to advanced patterns, using the production `circles-common-v2` microservice chart—specifically [deployment.yaml](file:///Users/wishwa.wijeratne/circles/sg/sre/helm-charts/circles-common-v2/templates/deployment.yaml) and [_helpers.tpl](file:///Users/wishwa.wijeratne/circles/sg/sre/helm-charts/circles-common-v2/templates/_helpers.tpl)—as a practical case study.

---

## Table of Contents
1. [Introduction to the Helm Templating Engine](#1-introduction-to-the-helm-templating-engine)
2. [Basic Syntax & Values Access](#2-basic-syntax--values-access)
3. [Helm Built-in Objects](#3-helm-built-in-objects)
4. [Whitespace Control & YAML Formatting](#4-whitespace-control--yaml-formatting)
5. [Pipelines & Template Functions](#5-pipelines--template-functions)
6. [Control Flow: Conditionals & Loops](#6-control-flow-conditionals--loops)
7. [The Dot (`.`) vs. Dollar (`$`) Scope Rules](#7-the-dot--vs-dollar--scope-rules)
8. [Named Templates & Helpers (`_helpers.tpl`)](#8-named-templates--helpers-_helperstpl)
9. [Step-by-Step Walkthrough of `deployment.yaml`](#9-step-by-step-walkthrough-of-deploymentyaml)
10. [Debugging & Testing Templates](#10-debugging--testing-templates)

---

## 1. Introduction to the Helm Templating Engine

Helm templates are Kubernetes manifest files written in **YAML**, embedded with Go's `text/template` engine and over 100+ helper functions from the **Sprig library**.

When you run `helm install` or `helm template`, Helm:
1. Loads the template files inside `templates/`.
2. Merges default `values.yaml` with environment value files and command-line `--set` overrides.
3. Renders the template actions (`{{ ... }}`) into plain, valid Kubernetes YAML.
4. Sends the final YAML manifests to the Kubernetes API server (`kubectl apply`).

---

## 2. Basic Syntax & Values Access

All template actions are enclosed within double curly braces `{{` and `}}`.

### Accessing Values
Values passed into the chart are accessed via `.Values`:

```yaml
# Input values.yaml:
# appName: "fulfillment-service"

# Template:
metadata:
  name: {{ .Values.appName }}

# Rendered Output:
metadata:
  name: fulfillment-service
```

### Accessing Nested Values
Use dot notation to navigate nested YAML structures:

```yaml
# Input values.yaml:
# externalSecret:
#   enabled: true

# Template:
{{- if .Values.externalSecret.enabled }}
```

---

## 3. Helm Built-in Objects

Helm automatically injects several top-level objects into every template context:

| Object | Description | Examples |
|---|---|---|
| `.Values` | Values passed from `values.yaml`, `-f valueFiles/*.yaml`, or `--set`. | `.Values.env`, `.Values.image` |
| `.Release` | Information about the current Helm release operation. | `.Release.Name`, `.Release.Namespace`, `.Release.IsUpgrade` |
| `.Chart` | Metadata declared inside `Chart.yaml`. | `.Chart.Name`, `.Chart.Version`, `.Chart.AppVersion` |
| `.Files` | Provides access to non-template files inside the chart. | `.Files.Get "config.json"`, `.Files.GetBytes` |
| `.Capabilities` | Information about Kubernetes cluster capabilities and API versions. | `.Capabilities.KubeVersion.Version` |

---

## 4. Whitespace Control & YAML Formatting

YAML is strictly whitespace-sensitive. Extra spaces or newline characters created by template blocks will cause YAML syntax errors.

### A. Stripping Whitespace (`{{-` and `-}}`)
* `{{-` removes all whitespace and newlines **to the left** of the tag.
* `-}}` removes all whitespace and newlines **to the right** of the tag.

#### Without whitespace control:
```yaml
items:
{{ if .Values.enabled }}
  - name: my-item
{{ end }}
```
*Rendered Output (contains unwanted blank lines):*
```yaml
items:

  - name: my-item

```

#### With whitespace control (`{{-` and `-}}`):
```yaml
items:
{{- if .Values.enabled }}
  - name: my-item
{{- end }}
```
*Rendered Output (clean):*
```yaml
items:
  - name: my-item
```

### B. Indentation Control (`indent` and `nindent`)
When injecting multiline strings or objects into a YAML template, use `nindent` (newline + indent) or `indent`:

```yaml
# Formats multi-line string or YAML block cleanly with 4 spaces of indentation:
data:
{{ toYaml .Values.customConfig | nindent 4 }}
```

---

## 5. Pipelines & Template Functions

Helm uses Unix-style pipeline operators (`|`) to chain functions together, passing the output of one function as the last argument to the next.

### Key Template Functions

#### 1. `quote` / `squote`
Wraps strings in double or single quotes. **Essential for numbers, booleans, or strings with special characters in YAML.**
```yaml
env:
  - name: PORT
    value: {{ .Values.containerPort | quote }}
```
*Output:* `value: "8080"`

#### 2. `default`
Provides a fallback value if the target variable is `nil` or empty `""`.
```yaml
replicas: {{ default 1 .Values.replicas }}
```
*If `.Values.replicas` is unset, output is `1`.*

#### 3. String Truncation & Cleaning (`trunc`, `trimSuffix`)
Used for generating valid Kubernetes resource names (max 63 characters, must not end with a hyphen `-`):
```yaml
{{ .Release.Name | trunc 63 | trimSuffix "-" }}
```

#### 4. File Path Helpers (`base`, `dir`)
```yaml
# Given filepath = "/etc/myapp/config.yml"
filename: {{ $filepath | base }}   # Output: "config.yml"
directory: {{ $filepath | dir }}    # Output: "/etc/myapp"
```

---

## 6. Control Flow: Conditionals & Loops

### A. Conditionals (`if`, `else if`, `else`)

```yaml
{{- if eq .Values.env "production" }}
replicas: 5
{{- else if eq .Values.env "staging" }}
replicas: 2
{{- else }}
replicas: 1
{{- end }}
```

#### Common Evaluation Operators:
* `eq`: Equal (`eq a b`)
* `ne`: Not equal (`ne a b`)
* `and`: Logical AND (`and a b`)
* `or`: Logical OR (`or a b`)
* `not`: Logical NOT (`not a`)

---

### B. Looping (`range`)

The `range` operator iterates over arrays (lists) or maps (key-value pairs).

#### Iterating Over a List:
```yaml
# Given values.yaml:
# configMaps:
#   - "/etc/app/config.yml"
#   - "/etc/app/db.yml"

{{- range $index, $filepath := .Values.configMaps }}
- name: config-volume-{{ $index }}
  mountPath: {{ $filepath }}
{{- end }}
```

#### Iterating Over a Key-Value Map:
```yaml
# Given values.yaml:
# extraEnv:
#   LOG_LEVEL: "debug"
#   ENABLE_FEATURE: "true"

{{- range $name, $value := .Values.extraEnv }}
- name: {{ $name }}
  value: {{ quote $value }}
{{- end }}
```

---

## 7. The Dot (`.`) vs. Dollar (`$`) Scope Rules

This is one of the most critical concepts in Helm templating.

### The Scope Swap Problem
When inside a `range` loop or a `with` block, the dot context `.` **changes** to represent the current iteration item rather than the root context!

```yaml
# BROKEN EXAMPLE:
{{- range $filepath := .Values.configMaps }}
# Inside here, '.' is now '$filepath' (a string), NOT the root object!
# So '.Values.appName' WILL FAIL because string doesn't have a '.Values' field!
name: {{ .Values.appName }}-config  # ❌ ERROR!
{{- end }}
```

### The Solution: Using `$` (Root Scope)
The `$` variable **always** points to the top-level root context, regardless of how deep you are inside loops or conditionals.

```yaml
# CORRECT EXAMPLE:
{{- range $index, $filepath := .Values.configMaps }}
# Use '$.Values' to safely access root values inside a loop:
name: {{ $.Values.appName }}-{{ $.Values.env }}-volume-{{ $index }} # ✅ SUCCESS!
{{- end }}
```

---

## 8. Named Templates & Helpers (`_helpers.tpl`)

Files named `_helpers.tpl` (or any file starting with `_`) are ignored when generating Kubernetes manifests. They are used to declare **reusable partial templates**.

### Defining a Helper ([_helpers.tpl](file:///Users/wishwa.wijeratne/circles/sg/sre/helm-charts/circles-common-v2/templates/_helpers.tpl))
```gotemplate
{{/*
Generates a truncated, clean release name.
*/}}
{{- define "fullname" -}}
    {{- printf "%s" .Release.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{/*
Maps environment name to regional environment prefix (e.g. staging + sg -> ssg).
*/}}
{{- define "envprefix" -}}
{{- if eq $.Values.env "staging" -}}
{{- printf "%s%s" "s" $.Values.region -}}
{{- else if eq $.Values.env "preproduction" -}}
{{- printf "%s%s" "q" $.Values.region -}}
{{- else if eq $.Values.env "production" -}}
{{- printf "%s%s" "p" $.Values.region -}}
{{- end -}}
{{- end -}}
```

### Invoking Named Templates

#### 1. Using `template` (Direct Insertion)
```yaml
metadata:
  name: {{ template "fullname" . }}
```
*Inserts rendered output directly into the document.*

#### 2. Using `include` (Pipeline Supported)
```yaml
env:
  - name: ENV_PREFIX
    value: {{ include "envprefix" $ | quote }}
```
*Returns output as a string so it can be passed to functions like `quote` or `nindent`.*

---

## 9. Step-by-Step Walkthrough of `deployment.yaml`

Let's break down the production Deployment manifest [deployment.yaml](file:///Users/wishwa.wijeratne/circles/sg/sre/helm-charts/circles-common-v2/templates/deployment.yaml) section by section.

### Section 1: Resource Metadata & CI Annotations
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ template "fullname" . }}
  annotations:
    DeploymentHash: {{ .Values.DeploymentHash | default "None" | quote }}
    BuildNumber: {{ .Values.BuildNumber | default "None" | quote }}
    Branch: {{ .Values.branch | default "None" | quote }}
    CommitId: {{ .Values.commitId | default "None" | quote }}
```
* **Explanation:** Uses `template "fullname" .` to set the K8s deployment name. Injects CI/CD build metadata via pipelines using `| default "None" | quote`.

---

### Section 2: Blue/Green Label Selectors
```yaml
  labels:
    release: {{ template "fullname" . }}
    app: {{ template "fullname" . }}
    env: {{ .Values.env }}
    region: {{ .Values.region }}
    slot: {{ .Values.slot }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      release: {{ .Values.appName }}
      app: {{ .Values.appName }}
      env: {{ .Values.env }}
      slot: {{ .Values.slot }}
```
* **Explanation:** Injects `env`, `region`, and `slot` (`blue` or `green`). Kubernetes Deployment selectors rely on these labels to manage active vs inactive deployment slots.

---

### Section 3: Conditional InitContainers (ExternalSecret / CTE Integration)
```yaml
      {{- if $.Values.externalSecret }}
      {{- if $.Values.externalSecret.enabled }}
      {{- if eq $.Values.configEnabled "true" }}
      initContainers:
      - name: {{ $.Values.appName }}-{{ $.Values.env }}-{{ $.Values.region }}-{{ $.Values.slot }}-configcontainer
        image: "{{ $.Values.initContainer.image }}:{{ $.Values.initContainer.imageTag }}"
        env:
        - name: envprefix
          value: {{ include "envprefix" $ }}
        command: ['sh', '-c', 'python envsubst.py']
      {{- end }}
      {{- end }}
      {{- end }}
```
* **Explanation:** Checks three conditions before injecting an initContainer. Uses `include "envprefix" $` to pass the generated prefix (e.g. `ssg`) into `envsubst.py` for dynamic secret injection.

---

### Section 4: Dynamic Volume & VolumeMount Processing
```yaml
      volumes:
      {{- if eq $.Values.configEnabled "true" }}
      {{- range $index, $filepath := .Values.configMaps }}
      - name: {{ $.Values.appName }}-{{ $.Values.env }}-{{ $.Values.region }}-{{ $.Values.slot }}-configvolume-{{ $index }}
        configMap:
          name: {{ $.Values.appName }}-{{ $.Values.env }}-{{ $.Values.region }}-{{ $.Values.slot }}-configmap-{{ $index }}
          items:
          - key: {{ $filepath | base }}
            path: {{ $filepath | base }}
      {{- end }}
      {{- end }}
```
* **Explanation:** Loops through `.Values.configMaps` array. Uses `$.Values` scope to construct unique volume names and `$filepath | base` to set target keys and filenames inside the ConfigMap.

---

### Section 5: Dynamic Extra Environment Variables
```yaml
          env:
            - name: appName
              value: {{ .Values.appName | quote }}
            - name: env
              value: {{ .Values.env | quote }}
          {{- range $name, $value := .Values.extraEnv }}
            - name: {{ $name }}
              value: {{ quote $value }}
          {{- end }}
```
* **Explanation:** Appends custom application environment variables from `extraEnv` map into the container specification cleanly.

---

### Section 6: Health Probes & Resource Limits
```yaml
          resources:
            requests:
              cpu: "{{default "100m" .Values.podCpuRequests}}"
              memory: "{{default "100Mi" .Values.podMemoryRequests}}"
            limits:
              memory: {{default "100Mi" .Values.podMemoryLimits }}
              cpu: {{default "100m" .Values.podCpuLimits }}
```
* **Explanation:** Uses `default` to provide sensible CPU and memory fallbacks if unconfigured in value files.

---

## 10. Debugging & Testing Templates

### 1. Linting
Check for syntax errors or invalid template logic:
```bash
helm lint circles-common-v2/ -f valueFiles/go-service/staging.yaml
```

### 2. Dry-Run Rendering (`helm template`)
Render and inspect the generated Kubernetes YAML without deploying to a cluster:
```bash
helm template test-release circles-common-v2/ \
  -f valueFiles/go-service/staging.yaml \
  --set appName=fulfillment-service \
  --set region=sg \
  --set slot=blue
```

### 3. Debugging Evaluation Errors (`--debug`)
If template rendering fails, add `--debug` to print the full stack trace and partial YAML output up to the point of failure:
```bash
helm template test-release circles-common-v2/ --debug
```
