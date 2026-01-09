## Workshop 15 — Ingress & HTTP Routing (Acceso Web Real en Kubernetes)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop entenderás **con claridad enterprise**:

- Por qué **Ingress ≠ Service**
- Qué problema real resuelve Ingress
- Qué es un **Ingress Controller**
- Flujo real de tráfico HTTP/HTTPS en Kubernetes
- Diferencia entre:
  - Service `LoadBalancer`
  - Ingress
- Cómo explicar **arquitectura web Kubernetes** en entrevistas
- Qué se usa **realmente en producción**

---

## Regla 0 — Contexto y estado base (OBLIGATORIO)

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

Verifica:

```
kubectl get ns practice
```

---

## Estado inicial requerido

Necesitamos una app expuesta vía **Service interno**.

Si no existe `web-app`:

```
kubectl -n practice create deployment web-app --image=nginx:1.27
kubectl -n practice scale deployment web-app --replicas=2
```

Crear Service ClusterIP:

```
kubectl -n practice expose deployment web-app \
  --name web-svc \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

Verifica:

```
kubectl -n practice get deploy,svc,pods
```

---

## Concepto clave (muy importante)

> **Ingress NO expone Pods**
>
> Ingress **expone Services**
>
> Ingress trabaja en **capa HTTP/HTTPS (L7)**

---

## Parte A — Qué es Ingress (sin humo)

Ingress es un **objeto Kubernetes** que define:

- Hosts (`example.com`)
- Paths (`/`, `/api`, `/app`)
- Reglas HTTP
- TLS

Pero **NO funciona solo**.

Necesita un **Ingress Controller**.

---

## Parte B — Ingress Controller (pieza crítica)

Un **Ingress Controller** es:

- Un controlador que:
  - Lee objetos Ingress
  - Configura un proxy real (NGINX, Traefik, HAProxy)

Ejemplos reales:
- NGINX Ingress Controller (más común)
- Traefik
- HAProxy
- Cloud-native (ALB Ingress Controller en AWS)

---

## Parte C — Instalar NGINX Ingress Controller (kind)

### C.1 Instalar controlador

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

Esto crea:
- Namespace `ingress-nginx`
- Pods del controller
- Service tipo NodePort

Verifica:

```
kubectl get pods -n ingress-nginx
```

Estado esperado:

```
controller   Running
```

---

## Parte D — Crear Ingress básico

---

## D.1 Crear archivo Ingress

Crea archivo `web-ingress.yaml`:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: practice
spec:
  rules:
  - host: web.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
```

Aplicar:

```
kubectl apply -f web-ingress.yaml
```

Verifica:

```
kubectl -n practice get ingress
```

---

## Parte E — Resolver DNS local (LAB)

En **kind**, debes simular DNS.

Edita `/etc/hosts` en la VM:

```
sudo nano /etc/hosts
```

Agrega:

```
127.0.0.1 web.local
```

---

## Parte F — Probar acceso HTTP

```
curl http://web.local
```

Resultado esperado:

```
<html>...</html>
```

---

## Flujo REAL de tráfico (mentalidad arquitectura)

```
Cliente
  |
  v
Ingress Controller (NGINX)
  |
  v
Service (ClusterIP)
  |
  v
Pods (Deployment)
```

---

## Parte G — Tabla clave (entrevista)

| Componente | Capa | Rol |
|---|---|---|
| Pod | L7 | Ejecuta app |
| Service | L4 | Balancea Pods |
| Ingress | L7 | Routing HTTP |
| Ingress Controller | L7 | Proxy real |

---

## Parte H — Por qué Ingress es PRODUCCIÓN REAL

- Un solo endpoint público
- Múltiples apps
- TLS centralizado
- Routing por host/path
- Costos menores (1 LB vs muchos)

---

## Parte I — Errores comunes

- ❌ Creer que Ingress reemplaza Services
- ❌ Exponer cada app con LoadBalancer
- ❌ No entender que Ingress necesita controller
- ❌ Pensar que Ingress funciona solo en kind

---

## Cómo explicarlo en entrevista (respuesta modelo)

> “En producción usamos Ingress porque permite exponer múltiples  
> aplicaciones HTTP bajo un solo endpoint, delegando balanceo L4  
> al Service y routing L7 al Ingress Controller, reduciendo costos  
> y mejorando gobernanza.”

---

## Cleanup (opcional)

```
kubectl -n practice delete ingress web-ingress
kubectl -n practice delete svc web-svc
kubectl -n practice delete deployment web-app
kubectl delete namespace ingress-nginx
```

---

