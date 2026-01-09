## Workshop 20 — Liveness & Readiness Probes (Salud real de aplicaciones)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué son Liveness y Readiness Probes**
- Diferenciar **cuándo usar cada una**
- Entender **cómo Kubernetes decide reiniciar o enrutar tráfico**
- Implementar probes HTTP y exec
- Identificar **errores comunes en producción**
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

> **Liveness Probe**
>
> “¿El contenedor sigue vivo?”
>
> Si falla → Kubernetes **reinicia** el contenedor

> **Readiness Probe**
>
> “¿Está listo para recibir tráfico?”
>
> Si falla → Kubernetes **saca el Pod del Service**,  
> pero **NO lo reinicia**

---

## Parte A — Deployment SIN probes (riesgo real)

Archivo: `deployment-no-probes.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-no-probes
  namespace: practice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-no-probes
  template:
    metadata:
      labels:
        app: web-no-probes
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
```

Aplicar:

```
kubectl apply -f deployment-no-probes.yaml
```

👉 Kubernetes **asume** que la app está bien, aunque no lo esté.

---

## Parte B — Readiness Probe (control de tráfico)

Archivo: `deployment-readiness.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-readiness
  namespace: practice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-readiness
  template:
    metadata:
      labels:
        app: web-readiness
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

Aplicar:

```
kubectl apply -f deployment-readiness.yaml
```

Ver estado:

```
kubectl -n practice get pods
```

---

## Parte C — Simular fallo de Readiness

Entrar a un Pod:

```
kubectl -n practice exec -it pod/<POD_NAME> -- sh
```

Romper endpoint:

```
rm /usr/share/nginx/html/index.html
```

Ver estado:

```
kubectl -n practice get pods
```

Resultado esperado:

```
READY: 0/1
STATUS: Running
```

👉 El Pod **NO se reinicia**, solo se marca como no listo.

---

## Parte D — Liveness Probe (reinicio automático)

Archivo: `deployment-liveness.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-liveness
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-liveness
  template:
    metadata:
      labels:
        app: web-liveness
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
```

Aplicar:

```
kubectl apply -f deployment-liveness.yaml
```

---

## Parte E — Simular fallo de Liveness

Entrar al Pod:

```
kubectl -n practice exec -it pod/<POD_NAME> -- sh
```

Romper nginx:

```
kill 1
```

Ver eventos:

```
kubectl -n practice get pods -w
```

Resultado esperado:

- Pod reiniciado automáticamente
- Restart count incrementa

---

## Parte F — Probes combinadas (patrón real)

Archivo: `deployment-full-probes.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-full-probes
  namespace: practice
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-full-probes
  template:
    metadata:
      labels:
        app: web-full-probes
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
```

👉 Patrón recomendado:

- Readiness primero
- Liveness después

---

## Parte G — Errores críticos en producción

❌ Errores comunes:

- No usar probes
- Usar liveness demasiado agresivo
- Usar misma lógica para readiness y liveness
- No considerar startup time

✅ Buenas prácticas:

- Readiness para tráfico
- Liveness para cuelgues reales
- `initialDelaySeconds` bien calibrado
- Probes simples y rápidas

---

## Parte H — Pregunta típica de entrevista

**Pregunta:**
> ¿Cuál es la diferencia entre Liveness y Readiness?

**Respuesta senior:**
> “Readiness controla el enrutamiento de tráfico,
> Liveness controla el ciclo de vida del contenedor.
> Readiness no reinicia Pods, Liveness sí.”

---

## Cleanup (opcional)

```
kubectl -n practice delete deployment web-no-probes web-readiness web-liveness web-full-probes
```

---
