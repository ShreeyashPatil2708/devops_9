# values.yaml → templates/ → rendered output

This shows exactly how a setting in `values.yaml` flows through a template file and
becomes real YAML that Kubernetes understands. All "rendered result" lines below were
taken directly from `outputs/rendered-default.yaml`, produced by the real
`helm template webrelease ./webapp -n demo` command (not invented).

## Example 1 — replica count

**values.yaml**
```yaml
replicaCount: 3
```

**templates/deployment.yaml**
```yaml
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
```

**Rendered result** (`outputs/rendered-default.yaml`)
```yaml
spec:
  replicas: 3
```

## Example 2 — container image

**values.yaml**
```yaml
image:
  repository: httpd
  pullPolicy: IfNotPresent
  tag: "2.4"
```

**templates/deployment.yaml**
```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
imagePullPolicy: {{ .Values.image.pullPolicy }}
```

**Rendered result**
```yaml
image: "httpd:2.4"
imagePullPolicy: IfNotPresent
```

## Example 3 — Service type and port

**values.yaml**
```yaml
service:
  type: ClusterIP
  port: 80
```

**templates/service.yaml**
```yaml
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
```

**Rendered result**
```yaml
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: http
```

## Proof that overrides also work (the `--set` flag)

Running:
```
helm template webrelease ./webapp -n demo --set replicaCount=5 --set image.tag=2.4.62
```
produced a different render. Comparing the two output files
(`diff outputs/rendered-default.yaml outputs/rendered-override.yaml`) gave this real
diff, saved in `outputs/step3-2-template-diff.txt`:

```
51c51
<   replicas: 3
---
>   replicas: 5
68c68
<           image: "httpd:2.4"
---
>           image: "httpd:2.4.62"
```

This proves two things:
1. `values.yaml` is the single source of default configuration for the templates.
2. `--set key=value` on the command line overrides values.yaml at render/install/upgrade
   time, without editing any file.
