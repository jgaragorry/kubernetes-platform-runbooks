## Workshop 14 — Services & Networking (Comunicación en Kubernetes)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop entenderás **sin ambigüedades**:

- Por qué los **Pods no deben exponerse directamente**
- Qué es un **Service** y qué problema resuelve
- Diferencias **reales y prácticas** entre:
  - `ClusterIP`
  - `NodePort`
  - `LoadBalancer`
- Cómo funciona el **DNS interno**
- Qué se usa en **labs**, **producción** y **cloud**
- Cómo explicarlo **con criterio enterprise en entrevistas**

---

## Regla 0 — Contexto y estado base (OBLIGATORIO)

Verifica contexto activo:

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

Si no existe:

```
kubectl create namespace practice
```

---

## Estado inicial requerido

Necesitamos un **Deployment estable** que represente una app real.

Si no existe `web-app`, créalo:

```
kubectl -n practice create deployment web-app --image=nginx:1.27
kubectl -n practice scale deployment web-app --replicas=2
```

Verifica:

```
kubectl -n practice get deploy,pods -o wide
```

---

## Concepto clave (mentalidad Kubernetes)

> **Los Pods son efímeros y sus IPs cambian**  
>  
> Nunca debes depender de la IP de un Pod  
>  
> **Services abstraen Pods**

---

## Parte A — ClusterIP (Service interno)

---

## A.1 Qué es ClusterIP

- Tipo **por defecto**
- Solo accesible **dentro del cluster**
- Ideal para:
  - Comunicación entre microservicios
  - Backends internos
  - APIs internas

---

## A.2 Crear Service ClusterIP

```
kubectl -n practice expose deployment web-app \
  --name web-svc \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

Verifica:

```
kubectl -n practice get svc
```

Resultado esperado:

```
NAME      TYPE        CLUSTER-IP     PORT(S)
web-svc   ClusterIP   10.x.x.x       80/TCP
```

---

## A.3 Probar acceso interno (DNS)

Crea un Pod temporal para probar conectividad:

```
kubectl -n practice run curl-pod \
  --image=curlimages/curl:8.5.0 \
  --restart=Never \
  -it --rm -- sh
```

Dentro del Pod ejecuta:

```
curl http://web-svc
```

Resultado esperado:

```
<html>...</html>
```

Salir:

```
exit
```

---

## Concepto clave (DNS interno)

Kubernetes crea automáticamente:

```
web-svc.practice.svc.cluster.local
```

Puedes probar:

```
curl http://web-svc.practice.svc.cluster.local
```

---

## Parte B — NodePort (exposición básica)

---

## B.1 Qué es NodePort

- Expone el Service en **cada nodo**
- Usa un puerto alto (30000–32767)
- No recomendado en producción cloud
- Útil para:
  - Labs
  - Debug
  - Demos rápidas

---

## B.2 Crear Service NodePort

Primero elimina el Service anterior:

```
kubectl -n practice delete svc web-svc
```

Crear NodePort:

```
kubectl -n practice expose deployment web-app \
  --name web-nodeport \
  --port 80 \
  --target-port 80 \
  --type NodePort
```

Verifica:

```
kubectl -n practice get svc web-nodeport
```

Resultado esperado:

```
PORT(S): 80:<NODEPORT>/TCP
```

---

## B.3 Acceder desde el host

Obtén IP del nodo:

```
kubectl get nodes -o wide
```

Accede desde tu VM:

```
curl http://<NODE-IP>:<NODEPORT>
```

---

## Concepto clave (entrevista)

> **NodePort no balancea tráfico de forma real**  
>  
> Solo abre un puerto en cada nodo

---

## Parte C — LoadBalancer (modo cloud)

---

## C.1 Qué es LoadBalancer

- Crea un **balanceador externo**
- Cloud-native
- Usado en:
  - AKS
  - EKS
  - GKE

En **kind NO funciona de forma real**, pero debes conocerlo.

---

## C.2 Crear Service LoadBalancer

```
kubectl -n practice delete svc web-nodeport
```

```
kubectl -n practice expose deployment web-app \
  --name web-lb \
  --port 80 \
  --target-port 80 \
  --type LoadBalancer
```

Verifica:

```
kubectl -n practice get svc web-lb
```

Resultado esperado en kind:

```
EXTERNAL-IP: <pending>
```

---

## Concepto clave (cloud)

En cloud real:
- AKS → Azure Load Balancer
- EKS → ELB / ALB
- GKE → Cloud Load Balancer

---

## Parte D — Tabla comparativa (MUY entrevista)

---

| Tipo | Acceso | Uso real |
|---|---|---|
| ClusterIP | Interno | Microservicios |
| NodePort | Nodo:Puerto | Labs / Debug |
| LoadBalancer | Público | Producción |
| Ingress | HTTP/HTTPS | Producción real |

---

## Parte E — Errores comunes

- ❌ Exponer Pods directamente
- ❌ Usar NodePort en producción
- ❌ No entender DNS interno
- ❌ Confundir Service con Pod

---

## Parte F — Cómo lo explicas en entrevista

> “Los Pods son efímeros, por eso Kubernetes introduce Services  
> que proveen un endpoint estable, desacoplado del ciclo de vida  
> de los Pods, permitiendo balanceo y descubrimiento por DNS.”

---

## Cleanup (opcional)

```
kubectl -n practice delete svc web-lb
kubectl -n practice delete deployment web-app
```

---
