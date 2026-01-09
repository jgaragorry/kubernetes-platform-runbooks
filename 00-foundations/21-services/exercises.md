## Workshop 21 — Services (ClusterIP, NodePort, LoadBalancer)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué los Pods NO deben exponerse directamente**
- Entender **qué problema resuelve un Service**
- Diferenciar **ClusterIP, NodePort y LoadBalancer**
- Saber **cuándo usar cada tipo**
- Diagnosticar errores comunes de conectividad
- Responder preguntas típicas de entrevistas (AKS / EKS / GKE)

---

## Regla 0 — Contexto y namespace

Verifica contexto:

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

> **Un Pod NO es estable**
>
> - IP cambia
> - Puede morir y recrearse
> - No es un endpoint confiable

👉 **Service = endpoint estable + balanceo**

---

## Parte A — Deployment base (app backend)

Archivo: `deployment-web.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: practice
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

Aplicar:

```
kubectl apply -f deployment-web.yaml
```

Ver Pods:

```
kubectl -n practice get pods -o wide
```

Observa:
- IPs distintas
- IPs internas (10.x.x.x)

---

## Parte B — Service ClusterIP (default)

### ¿Qué es ClusterIP?

> Acceso **solo dentro del cluster**
>
> Backend ↔ Backend

Archivo: `svc-clusterip.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: web-clusterip
  namespace: practice
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

Aplicar:

```
kubectl apply -f svc-clusterip.yaml
```

Ver Service:

```
kubectl -n practice get svc
```

Resultado esperado:

- ClusterIP asignada (10.96.x.x)
- Endpoint estable

---

## Parte C — Probar ClusterIP desde dentro

Crear Pod de pruebas:

```
kubectl -n practice run curl-test --image=curlimages/curl --restart=Never -it -- sh
```

Desde el Pod:

```
curl http://web-clusterip
```

👉 Funciona porque:
- DNS interno
- kube-proxy
- balanceo automático

Salir:

```
exit
```

---

## Parte D — NodePort (exposición básica)

### ¿Qué es NodePort?

> Expone el Service en:
>
> `<NODE_IP>:30000-32767`

Archivo: `svc-nodeport.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
  namespace: practice
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

Aplicar:

```
kubectl apply -f svc-nodeport.yaml
```

Ver Service:

```
kubectl -n practice get svc web-nodeport
```

---

## Parte E — Probar NodePort (kind)

En kind, obtener IP del nodo:

```
kubectl get nodes -o wide
```

O usar port-forward (recomendado en local):

```
kubectl -n practice port-forward svc/web-nodeport 8080:80
```

Probar:

```
curl http://localhost:8080
```

---

## Parte F — LoadBalancer (conceptual)

### ¿Qué es LoadBalancer?

> Service integrado con **cloud provider**
>
> - AKS → Azure Load Balancer
> - EKS → ELB/NLB
> - GKE → Cloud LB

Archivo: `svc-loadbalancer.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  namespace: practice
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
```

Aplicar (solo para ver estado):

```
kubectl apply -f svc-loadbalancer.yaml
```

Ver:

```
kubectl -n practice get svc web-lb
```

En kind:
- EXTERNAL-IP = pending (normal)

---

## Parte G — Tabla comparativa (clave entrevistas)

| Tipo | Acceso | Uso típico | Producción |
|----|----|----|----|
| ClusterIP | Interno | Backend ↔ Backend | ✅ Siempre |
| NodePort | Básico | Labs / debug | ⚠️ Limitado |
| LoadBalancer | Externo | Apps públicas | ✅ Cloud |

---

## Parte H — Errores comunes

❌ Errores:

- Exponer Pods directamente
- Usar NodePort en producción
- No alinear selector con labels
- Confundir port vs targetPort

✅ Buenas prácticas:

- Services siempre delante de Pods
- ClusterIP por defecto
- LoadBalancer solo en cloud
- Labels claros y consistentes

---

## Parte I — Pregunta típica de entrevista

**Pregunta:**
> ¿Por qué no accedemos directamente a Pods?

**Respuesta senior:**
> “Porque los Pods son efímeros.  
> El Service provee un endpoint estable, balanceo y desacopla la app de la infraestructura.”

---

## Cleanup (opcional)

```
kubectl -n practice delete svc web-clusterip web-nodeport web-lb
kubectl -n practice delete deployment web-app
kubectl -n practice delete pod curl-test
```

---
