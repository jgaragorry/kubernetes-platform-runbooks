## Workshop 29 — Requests & Limits (CPU / Memory)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué son Requests y Limits**
- Entender **por qué Kubernetes mata Pods (OOMKill)**
- Configurar recursos correctamente
- Responder preguntas reales de **SRE / AKS / EKS / producción**

---

## Regla 0 — Contexto y namespace

Verifica contexto activo:

```
kubectl config current-context
```

Resultado esperado:

```
kind-k8s-lab
```

Namespace de trabajo:

```
practice
```

Si no existe:

```
kubectl create namespace practice
```

---

## Concepto clave (muy importante)

> **Requests = lo que el Pod pide**  
> **Limits = lo máximo que el Pod puede usar**

Kubernetes **agenda por Requests**,  
pero **mata por Limits**.

---

## Diagrama mental (entrevista)

```
Node
 ├── CPU
 ├── Memory
 └── Pods
      ├── request → scheduling
      └── limit   → enforcement
```

---

## Parte A — Pod SIN requests ni limits (mala práctica)

Crear Pod simple:

```
kubectl -n practice run no-limits \
  --image=nginx:1.27 \
  --restart=Never
```

Ver:

```
kubectl -n practice describe pod no-limits
```

Observa:

- No hay requests
- No hay limits

❌ En producción esto es **riesgo alto**

---

## Parte B — Deployment con Requests y Limits

Crear archivo:

```
vi deploy-resources.yaml
```

Contenido:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-resources
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-resources
  template:
    metadata:
      labels:
        app: web-resources
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
```

Aplicar:

```
kubectl apply -f deploy-resources.yaml
```

---

## Parte C — Ver recursos asignados

```
kubectl -n practice describe pod <POD_NAME>
```

Busca:

```
Requests:
  cpu: 100m
  memory: 128Mi
Limits:
  cpu: 500m
  memory: 256Mi
```

---

## Parte D — Unidades (pregunta típica)

### CPU

- `100m` = 0.1 core
- `500m` = 0.5 core
- `1` = 1 core

### Memory

- `128Mi`
- `256Mi`
- `1Gi`

⚠️ No mezclar `M` con `Mi` en producción

---

## Parte E — Simular OOMKill (muy importante)

Crear Pod que consuma memoria:

```
vi oom.yaml
```

Contenido:

```
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
  namespace: practice
spec:
  containers:
  - name: stress
    image: polinux/stress
    args: ["--vm", "1", "--vm-bytes", "300M", "--vm-hang", "1"]
    resources:
      limits:
        memory: "200Mi"
```

Aplicar:

```
kubectl apply -f oom.yaml
```

Ver estado:

```
kubectl -n practice get pods
```

Describe:

```
kubectl -n practice describe pod oom-demo
```

Resultado esperado:

```
Reason: OOMKilled
```

✔ Kubernetes mató el contenedor por exceder el limit

---

## Parte F — Requests vs Limits (tabla mental)

```
Requests → scheduling
Limits   → enforcement
```

- Requests altos → menos Pods por nodo
- Limits bajos → OOMKill

---

## Parte G — QoS Classes (entrevista)

Kubernetes clasifica Pods:

```
Guaranteed
Burstable
BestEffort
```

### Guaranteed
- Requests == Limits
- Máxima prioridad

### Burstable
- Requests < Limits
- Caso más común

### BestEffort
- Sin requests ni limits
- Primeros en morir

Ver QoS:

```
kubectl -n practice describe pod <POD_NAME> | grep QoS
```

---

## Parte H — Buenas prácticas enterprise

✔ Siempre definir resources  
✔ Empezar conservador  
✔ Medir con metrics-server / Prometheus  
✔ Ajustar con HPA / VPA  

❌ Nunca dejar BestEffort en producción

---

## Parte I — Errores comunes

❌ Pensar que limit reserva recursos  
❌ No entender OOMKill  
❌ Usar valores arbitrarios  

Checklist debug:

```
kubectl top pod
kubectl describe pod
kubectl get events
```

---

## Parte J — Preguntas típicas de entrevista

- ¿Qué pasa si no defino requests?
- ¿Qué pasa si excedo memory?
- ¿CPU limit mata el Pod?
- ¿Qué es QoS Guaranteed?

Este lab te da las respuestas.

---

## Parte K — Cleanup

```
kubectl -n practice delete pod no-limits oom-demo
kubectl -n practice delete deployment web-resources
```

---
