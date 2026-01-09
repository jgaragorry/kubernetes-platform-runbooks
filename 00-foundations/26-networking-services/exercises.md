## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué los Pods no son accesibles directamente**
- Diferenciar **ClusterIP, NodePort y LoadBalancer**
- Entender el **modelo de networking interno de Kubernetes**
- Saber **cuándo usar cada tipo de Service**
- Responder preguntas típicas de entrevistas (AKS / EKS / GKE)

---

## Regla 0 — Contexto y namespace

Verifica contexto:

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

> **Los Pods son efímeros y tienen IP dinámica**

- IP del Pod cambia si el Pod muere
- Kubernetes **NO expone Pods directamente**
- El acceso se hace vía **Service**

---

## Diagrama mental (entrevista)

```
Cliente
   |
Service (IP estable)
   |
Selector
   |
Pods (IPs dinámicas)
```

---

## Parte A — Deployment base

Crear deployment:

```
kubectl -n practice create deployment web-app \
  --image=nginx:1.27
```

Escalar:

```
kubectl -n practice scale deployment web-app --replicas=2
```

Verificar:

```
kubectl -n practice get deploy,rs,pods -o wide
```

---

## Parte B — Service ClusterIP (default)

### Crear Service

```
kubectl -n practice expose deployment web-app \
  --name web-clusterip \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

Verificar:

```
kubectl -n practice get svc
```

Salida esperada:

```
web-clusterip   ClusterIP   10.x.x.x   <none>   80/TCP
```

---

## Parte C — Probar acceso interno

Crear Pod cliente:

```
kubectl -n practice run curl-pod \
  --image=curlimages/curl \
  --restart=Never \
  -it -- sh
```

Desde el Pod:

```
curl http://web-clusterip
exit
```

✔ Accesible **solo dentro del cluster**

---

## Parte D — Service NodePort

### Concepto

> Expone el servicio en **un puerto del nodo**

Rango típico:

```
30000–32767
```

---

### Crear NodePort

```
kubectl -n practice expose deployment web-app \
  --name web-nodeport \
  --port 80 \
  --target-port 80 \
  --type NodePort
```

Verificar:

```
kubectl -n practice get svc web-nodeport
```

Salida esperada:

```
NodePort  80:<nodePort>/TCP
```

---

### Acceso desde host

Obtener IP del nodo:

```
kubectl get nodes -o wide
```

Probar:

```
curl http://<NODE_IP>:<NODEPORT>
```

✔ Acceso externo funcional

---

## Parte E — Service LoadBalancer (conceptual)

### Concepto (cloud)

| Plataforma | Qué hace |
|----------|----------|
| AKS | Azure Load Balancer |
| EKS | AWS ELB |
| GKE | GCP Load Balancer |

En kind **no crea LB real**, queda en `pending`.

---

### Crear LoadBalancer

```
kubectl -n practice expose deployment web-app \
  --name web-lb \
  --port 80 \
  --target-port 80 \
  --type LoadBalancer
```

Ver:

```
kubectl -n practice get svc web-lb
```

Resultado esperado:

```
EXTERNAL-IP: <pending>
```

✔ Comportamiento correcto en local

---

## Parte F — Comparación (entrevista)

| Tipo | Acceso | Uso |
|----|------|----|
| ClusterIP | interno | microservicios |
| NodePort | externo básico | labs / debug |
| LoadBalancer | externo prod | apps públicas |

---

## Parte G — Cómo Kubernetes enruta tráfico

- Service crea **IP virtual**
- kube-proxy programa reglas (iptables / IPVS)
- balanceo **round-robin**

---

## Parte H — Errores comunes

❌ No responde el Service

Checklist:

```
kubectl get endpoints web-clusterip
kubectl describe svc web-clusterip
```

Causas:
- selector incorrecto
- Pods no Ready
- puerto mal definido

---

## Parte I — Buenas prácticas enterprise

✔ Nunca:
- exponer NodePort en producción

✔ Usar:
- ClusterIP + Ingress
- LoadBalancer solo con control

✔ Seguridad:
- NetworkPolicies
- TLS termination (Ingress)

---

## Parte J — Cleanup (opcional)

```
kubectl -n practice delete svc web-clusterip web-nodeport web-lb
kubectl -n practice delete deployment web-app
kubectl -n practice delete pod curl-pod
```

---
