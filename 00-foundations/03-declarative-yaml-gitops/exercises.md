## Workshop 4 — Kubernetes Declarativo (YAML) + GitOps Mindset (apply/diff/idempotencia)

### Objetivo (lo que vas a dominar)
Al finalizar este workshop vas a poder:

- Crear recursos **declarativamente** con YAML (sin comandos “imperativos” tipo `kubectl create ...`)
- Aplicar cambios con `kubectl apply` (idempotente)
- Previsualizar cambios con `kubectl diff` (control de cambios estilo GitOps)
- Entender y explicar:
  - `metadata.name`, `metadata.namespace`
  - labels/selector (Service → Pods)
  - replicas (Deployment)
- Troubleshooting: “no hay endpoints”, “namespace no existe”, “contexto equivocado”
- Cleanup seguro y repetible

---

## Reglas del módulo (para no perderte)

### Regla 0 — Verifica contexto SIEMPRE
Antes de tocar cualquier YAML:

```
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Si algo falla: **STOP**.

### Regla 1 — Un workspace por módulo
Usaremos:

- namespace: `practice`
- app label: `app=web-app`
- deployment: `web-app`
- service: `web-svc`

### Regla 2 — “Declarativo” = el repo es la fuente de verdad
- Tú editas YAML (deseado)
- `kubectl apply` converge el cluster al estado deseado

---

## Estructura recomendada dentro del repo

Crea esta carpeta y archivos:

```
00-foundations/03-declarative-yaml-gitops/
├── exercises.md
└── manifests/
    ├── 00-namespace.yaml
    ├── 10-deployment.yaml
    ├── 20-service.yaml
    └── kustomization.yaml   (opcional, recomendado)
```

> Nota: El orden numérico ayuda a ejecutarlo “sin pensar”.

---

## Paso 0 — Crear los manifests (YAML)

### 0.1 `manifests/00-namespace.yaml`

```
apiVersion: v1
kind: Namespace
metadata:
  name: practice
```

---

### 0.2 `manifests/10-deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: practice
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

**Qué NO debes romper aquí (estándar):**
- `spec.selector.matchLabels` debe coincidir con `spec.template.metadata.labels`
- Si no coinciden: el Deployment/RS se comportan raro y/o no gestionan pods como esperas

**Qué es personalizable:**
- `replicas` (2 hoy, mañana 3)
- `image` (nginx:1.27 hoy, mañana tu app)
- labels (pero deben permanecer consistentes)

---

### 0.3 `manifests/20-service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: web-svc
  namespace: practice
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - name: http
      port: 80
      targetPort: 80
```

**Estándar que no cambia:**
- `selector` debe apuntar a labels reales de los pods
- `type: ClusterIP` = exposición interna

**Personalizable:**
- `port` (el puerto del servicio)
- `targetPort` (el puerto real del contenedor)

---

### 0.4 (Opcional recomendado) `manifests/kustomization.yaml`

> Esto te da un “entrypoint” único estilo GitOps.

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - 00-namespace.yaml
  - 10-deployment.yaml
  - 20-service.yaml
```

---

## Paso 1 — Apply (crear/actualizar) de forma declarativa

### 1.1 Apply en orden (sin kustomize)

Desde la carpeta del repo (o ajusta la ruta):

```
kubectl apply -f 00-foundations/03-declarative-yaml-gitops/manifests/00-namespace.yaml
kubectl apply -f 00-foundations/03-declarative-yaml-gitops/manifests/10-deployment.yaml
kubectl apply -f 00-foundations/03-declarative-yaml-gitops/manifests/20-service.yaml
```

**Resultados esperados:**
- `namespace/practice created` (o `unchanged`)
- `deployment.apps/web-app created` (o `configured/unchanged`)
- `service/web-svc created` (o `configured/unchanged`)

### 1.2 Apply “GitOps style” con kustomize (recomendado)

Si tienes `kustomization.yaml`:

```
kubectl apply -k 00-foundations/03-declarative-yaml-gitops/manifests
```

**Resultado esperado:**
- Aplica los 3 recursos de una sola vez (idempotente)

---

## Paso 2 — Validación (lo mínimo que siempre debes ejecutar)

### 2.1 Ver recursos

```
kubectl -n practice get deploy,rs,pods,svc -o wide
```

**Esperado:**
- Deployment `web-app` READY 2/2
- 2 pods Running
- Service `web-svc` ClusterIP con IP asignada

### 2.2 Ver endpoints (confirmación real de ruteo)

```
kubectl -n practice get endpoints web-svc -o wide
```

**Esperado:**
- 2 endpoints con IP:80 (IPs de pods)

---

## Paso 3 — Diff (previsualizar cambios antes de aplicar)

> Esto es clave para entrevistas: “yo no aplico a ciegas”.

### 3.1 Cambia algo (ej: replicas 2 → 3)

Edita `manifests/10-deployment.yaml`:

- `replicas: 2` → `replicas: 3`

### 3.2 Ver diff

Con manifests directos:

```
kubectl diff -f 00-foundations/03-declarative-yaml-gitops/manifests/10-deployment.yaml || true
```

Con kustomize:

```
kubectl diff -k 00-foundations/03-declarative-yaml-gitops/manifests || true
```

**Notas importantes:**
- `kubectl diff` puede retornar exit code 1 cuando hay diferencias → por eso `|| true`
- Esto es normal en pipelines

### 3.3 Aplicar el cambio

```
kubectl apply -k 00-foundations/03-declarative-yaml-gitops/manifests
```

### 3.4 Ver convergencia (3 pods)

```
kubectl -n practice get pods
```

**Esperado:**
- 3 pods Running

---

## Paso 4 — Idempotencia real (la prueba de “ojos cerrados”)

Ejecuta apply otra vez sin cambiar nada:

```
kubectl apply -k 00-foundations/03-declarative-yaml-gitops/manifests
```

**Resultado esperado:**
- `unchanged` (o `configured` sin cambios reales)

**Interpretación clave:**
- Declarativo = repetible N veces sin romper nada

---

## Paso 5 — “Autocuración” (validación práctica)

Borra un pod y mira cómo vuelve (Deployment mantiene desired state):

1) Lista pods:

```
kubectl -n practice get pods
```

2) Borra uno:

```
kubectl -n practice delete pod/<POD_NAME>
```

3) Observa regeneración:

```
kubectl -n practice get pods -w
```

**Esperado:**
- vuelve a la cantidad de réplicas deseadas (2 o 3 según dejaste)

---

## Cleanup (declarativo, seguro y repetible)

### Opción A — Borrar por kustomize

```
kubectl delete -k 00-foundations/03-declarative-yaml-gitops/manifests --ignore-not-found=true
```

> Ojo: esto borra recursos definidos en kustomization, pero **no siempre** borra el namespace si tiene recursos externos. Aquí debería, porque todo está dentro.

### Opción B — Borrar namespace completo (borrón y cuenta nueva)

```
kubectl delete namespace practice --ignore-not-found=true
```

**Resultado esperado:**
- el namespace se elimina y con él todo lo que tenga dentro

---

## Troubleshooting (exactamente lo que te pasó antes)

### T1) “namespaces "practice" not found”
Causas típicas:
- no aplicaste `00-namespace.yaml`
- cambiaste de contexto

Diagnóstico:

```
kubectl config current-context
kubectl get ns | grep practice || true
```

Solución (si falta namespace):

```
kubectl apply -f 00-foundations/03-declarative-yaml-gitops/manifests/00-namespace.yaml
```

---

### T2) “Unable to connect to the server ... eks.amazonaws.com ...”
Causa: contexto equivocado o cluster inaccesible desde esa VM/red.

Solución:

1) Lista contextos:

```
kubectl config get-contexts
```

2) Cambia al correcto (ejemplo kind):

```
kubectl config use-context kind-k8s-lab
```

3) Confirma:

```
kubectl cluster-info
kubectl get nodes
```

---

### T3) Service sin endpoints (vacío)
Diagnóstico en orden:

1) ¿pods Ready?

```
kubectl -n practice get pods -o wide
```

2) ¿endpoints?

```
kubectl -n practice get endpoints web-svc -o wide
```

3) ¿selector del service coincide con labels de pods?

```
kubectl -n practice describe svc web-svc
kubectl -n practice get pods --show-labels
```

**Causa típica:**
- `selector.app` del service no coincide con `labels.app` de los pods

Solución:
- alinea `app: web-app` en `deployment.yaml` y `service.yaml`, luego:

```
kubectl apply -k 00-foundations/03-declarative-yaml-gitops/manifests
```

---

## Guion de entrevista (muy corto, pero demoledor)

- “Trabajo declarativo: el repo define el desired state.”
- “Antes de aplicar, uso `kubectl diff` para ver cambios.”
- “`kubectl apply` es idempotente: puedo repetirlo sin romper.”
- “Valido service por endpoints, no por intuición.”
- “Siempre verifico el contexto para evitar operar en el cluster equivocado.”

---

