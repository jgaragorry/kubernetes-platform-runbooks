## Workshop 30 — Health Checks (liveness, readiness, startup)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué un Pod “está Running pero no sirve”**
- Diferenciar **liveness, readiness y startup**
- Evitar **reinicios infinitos**
- Diseñar **deployments resilientes**
- Responder preguntas reales de **SRE / AKS / EKS**

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

## Concepto clave (entrevista)

> **Health checks NO son monitoreo**  
> Son **decisiones automáticas de Kubernetes**

---

## Diagrama mental

```
Container
 ├── liveness   → ¿estás vivo?
 ├── readiness  → ¿puedes recibir tráfico?
 └── startup    → ¿ya terminaste de arrancar?
```

---

## Parte A — Deployment SIN health checks (riesgo)

Crear deployment básico:

```
kubectl -n practice create deployment web-basic --image=nginx:1.27
```

Escalar:

```
kubectl -n practice scale deployment web-basic --replicas=1
```

Observa:

```
kubectl -n practice describe pod
```

No hay probes → Kubernetes **asume que todo está bien**

❌ En producción esto es peligroso

---

## Parte B — livenessProbe (reinicio automático)

Crear archivo:

```
vi deploy-liveness.yaml
```

Contenido:

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
kubectl apply -f deploy-liveness.yaml
```

---

## Parte C — Romper la app (simular fallo)

Entrar al Pod:

```
kubectl -n practice exec -it <POD_NAME> -- /bin/sh
```

Borrar contenido web:

```
rm -rf /usr/share/nginx/html/*
exit
```

Ver eventos:

```
kubectl -n practice describe pod <POD_NAME>
```

Resultado esperado:

```
Liveness probe failed
Killing container
```

✔ Kubernetes reinicia el Pod

---

## Parte D — readinessProbe (tráfico controlado)

Concepto clave:

> **Readiness NO reinicia el Pod**  
> Solo lo saca del Service

Crear archivo:

```
vi deploy-readiness.yaml
```

Contenido:

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
kubectl apply -f deploy-readiness.yaml
```

---

## Parte E — Ver endpoints listos

Crear Service:

```
kubectl -n practice expose deployment web-readiness \
  --name web-svc \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

Ver endpoints:

```
kubectl -n practice get endpoints web-svc
```

---

## Parte F — Simular Pod no-ready

Entrar a un Pod:

```
kubectl -n practice exec -it <POD_NAME> -- /bin/sh
```

Romper readiness:

```
rm -rf /usr/share/nginx/html/*
exit
```

Ver endpoints otra vez:

```
kubectl -n practice get endpoints web-svc
```

✔ El Pod sale del Service  
✔ No se reinicia

---

## Parte G — startupProbe (apps lentas)

Problema real:

- Apps Java
- Apps legacy
- Inicialización lenta

Sin startupProbe → **reinicios infinitos**

Crear archivo:

```
vi deploy-startup.yaml
```

Contenido:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-startup
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-startup
  template:
    metadata:
      labels:
        app: web-startup
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        startupProbe:
          httpGet:
            path: /
            port: 80
          failureThreshold: 30
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 10
```

Aplicar:

```
kubectl apply -f deploy-startup.yaml
```

---

## Parte H — Diferencias críticas (tabla)

```
liveness  → reinicia Pod
readiness → saca del Service
startup   → protege arranque
```

---

## Parte I — Buenas prácticas enterprise

✔ Usar readiness SIEMPRE  
✔ startup para apps lentas  
✔ liveness conservador  
✔ No copiar valores a ciegas  

❌ No usar probes = riesgo producción

---

## Parte J — Errores comunes

❌ readiness pensando que reinicia  
❌ liveness muy agresivo  
❌ no usar startupProbe  

Debug checklist:

```
kubectl describe pod
kubectl get events
kubectl logs
```

---

## Parte K — Preguntas típicas de entrevista

- ¿Por qué un Pod está Running pero no responde?
- ¿Cuándo usar startupProbe?
- ¿Readiness reinicia el Pod?
- ¿Qué pasa si falla liveness?

---

## Parte L — Cleanup

```
kubectl -n practice delete deployment web-basic web-liveness web-readiness web-startup
kubectl -n practice delete svc web-svc
```

---
