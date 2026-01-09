## Workshop 19 — Requests & Limits (CPU / Memory / Scheduling)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué son Requests y Limits**
- Entender **cómo Kubernetes decide dónde ejecutar un Pod**
- Diferenciar **CPU vs Memory** en Kubernetes
- Identificar causas reales de **OOMKilled**
- Relacionar Requests/Limits con **costos y estabilidad**
- Responder preguntas típicas de entrevistas (AKS / EKS / GKE)

---

## Regla 0 — Contexto correcto

```
kubectl config current-context
kubectl get nodes
```

Contexto esperado:

```
kind-k8s-lab
```

Namespace de trabajo:

```
practice
```

---

## Concepto clave (muy importante)

> **Requests = reserva**
>
> **Limits = límite máximo**
>
> Kubernetes:
> - **schedulea Pods usando Requests**
> - **mata contenedores usando Limits**

---

## Parte A — Requests vs Limits (teoría clara)

### CPU

- Unidad: `millicores (m)`
- 1000m = 1 core

Ejemplos:

```
cpu: "250m"   # 0.25 core
cpu: "1"      # 1 core
```

### Memory

- Unidad: Mi / Gi
- No se puede “throttlear” memoria → se mata el contenedor

Ejemplos:

```
memory: "128Mi"
memory: "512Mi"
```

---

## Parte B — Pod SIN requests ni limits (mala práctica)

Archivo: `pod-no-limits.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: no-limits-pod
  namespace: practice
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "256M"]
```

Aplicar:

```
kubectl apply -f pod-no-limits.yaml
```

Ver estado:

```
kubectl -n practice get pod no-limits-pod
```

👉 Este Pod **puede consumir todo el nodo**.

---

## Parte C — Pod con Requests y Limits (buena práctica)

Archivo: `pod-with-limits.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: with-limits-pod
  namespace: practice
spec:
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "256M"]
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "200Mi"
```

Aplicar:

```
kubectl apply -f pod-with-limits.yaml
```

---

## Parte D — Observar OOMKilled (memoria)

Ver estado:

```
kubectl -n practice get pod with-limits-pod
```

Describe:

```
kubectl -n practice describe pod with-limits-pod
```

Salida esperada:

```
State:     Terminated
Reason:    OOMKilled
```

👉 Kubernetes mató el contenedor por **exceder el límite de memoria**.

---

## Parte E — CPU Throttling (concepto clave)

CPU:

- **NO mata el contenedor**
- Reduce ejecución (throttle)

No verás errores, solo lentitud.

---

## Parte F — Requests afectan el Scheduler

### Ver Requests agregados

```
kubectl -n practice describe pod with-limits-pod | grep -A5 Requests
```

Scheduler decide:

> “¿Hay un nodo con al menos esta CPU y memoria disponibles?”

Si no → `Pending`

---

## Parte G — Default Requests (LimitRange)

Concepto enterprise:

- Evita Pods sin límites
- Forzado por plataforma

Ejemplo (NO aplicar aún):

```
kind: LimitRange
spec:
  limits:
  - default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
```

---

## Parte H — Requests/Limits en Deployments

Archivo: `deployment-limits.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-limited
  namespace: practice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-limited
  template:
    metadata:
      labels:
        app: web-limited
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
```

Aplicar:

```
kubectl apply -f deployment-limits.yaml
```

---

## Parte I — Errores comunes (entrevistas)

❌ Errores:

- No definir requests
- Limits demasiado bajos
- Copiar valores sin entender
- Usar mismos valores para todos

✅ Buenas prácticas:

- Requests realistas
- Limits con margen
- Ajustar con métricas (HPA)
- Separar entornos

---

## Parte J — Pregunta típica de entrevista

**Pregunta:**
> ¿Por qué un Pod entra en OOMKilled?

**Respuesta senior:**
> “Porque excedió su límite de memoria. Kubernetes no puede throttlear memoria,
> por lo que mata el contenedor para proteger el nodo.”

---

## Cleanup (opcional)

```
kubectl -n practice delete pod no-limits-pod with-limits-pod
kubectl -n practice delete deployment web-limited
```

---
