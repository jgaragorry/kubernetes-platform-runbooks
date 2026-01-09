## Workshop 27 — Ingress: Routing HTTP/HTTPS (Base Producción)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué Ingress existe**
- Diferenciar **Service vs Ingress**
- Entender el rol del **Ingress Controller**
- Implementar **routing por path**
- Responder preguntas reales de **AKS / EKS / GKE**

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

## Concepto clave (entrevista)

> **Ingress NO funciona solo**

Ingress es:
- una **regla**
- que necesita un **Ingress Controller**

Ejemplos de controllers:
- NGINX Ingress
- Traefik
- HAProxy
- Azure Application Gateway Ingress Controller (AKS)

---

## Diagrama mental (muy importante)

```
Cliente
  |
Ingress (HTTP routing)
  |
Service (ClusterIP)
  |
Pods
```

---

## Parte A — Preparar aplicaciones (2 backends)

### App 1 — web-app

```
kubectl -n practice create deployment web-app \
  --image=nginx:1.27
```

```
kubectl -n practice scale deployment web-app --replicas=2
```

```
kubectl -n practice expose deployment web-app \
  --name web-svc \
  --port 80 \
  --target-port 80
```

---

### App 2 — api-app

```
kubectl -n practice create deployment api-app \
  --image=nginx:1.27
```

```
kubectl -n practice scale deployment api-app --replicas=2
```

```
kubectl -n practice expose deployment api-app \
  --name api-svc \
  --port 80 \
  --target-port 80
```

Verificar:

```
kubectl -n practice get deploy,svc,pods
```

---

## Parte B — Instalar Ingress Controller (NGINX)

### Crear namespace del controller

```
kubectl create namespace ingress-nginx
```

---

### Instalar NGINX Ingress (manifest oficial)

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.0/deploy/static/provider/kind/deploy.yaml
```

---

### Esperar readiness

```
kubectl -n ingress-nginx get pods -w
```

Esperar:

```
ingress-nginx-controller   Running
```

---

## Parte C — Verificar Service del Ingress

```
kubectl -n ingress-nginx get svc
```

Resultado típico:

```
ingress-nginx-controller   NodePort
```

✔ En kind se expone vía NodePort

---

## Parte D — Crear recurso Ingress

### Ingress por path

Crear archivo:

```
vi ingress-paths.yaml
```

Contenido:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: practice-ingress
  namespace: practice
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-svc
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-svc
            port:
              number: 80
```

Aplicar:

```
kubectl apply -f ingress-paths.yaml
```

---

## Parte E — Probar routing

### Obtener puerto del Ingress

```
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

Busca:

```
80:<NODEPORT>
```

---

### Probar desde host

```
curl http://localhost:<NODEPORT>/web
```

```
curl http://localhost:<NODEPORT>/api
```

✔ Ambos deben responder HTML nginx

---

## Parte F — Qué aprendiste (entrevista)

✔ Ingress:
- no expone Pods
- enruta tráfico HTTP
- centraliza acceso

✔ Arquitectura real:
- Ingress
- Services internos
- Pods desacoplados

---

## Parte G — Errores comunes

❌ Ingress no responde

Checklist:

```
kubectl get ingress -n practice
kubectl describe ingress practice-ingress
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller
```

Causas:
- controller no instalado
- ingressClass incorrecta
- service mal referenciado

---

## Parte H — Buenas prácticas enterprise

✔ Usar Ingress para:
- routing
- TLS
- auth básica
- rate limit

✔ No usar:
- NodePort directo en producción

✔ En cloud:
- AKS: App Gateway Ingress
- EKS: ALB Ingress
- GKE: GCE Ingress

---

## Parte I — Preguntas típicas de entrevista

- ¿Ingress expone Pods?
- ¿Qué diferencia hay entre Service y Ingress?
- ¿Ingress funciona sin controller?
- ¿Dónde se configura TLS?

Debes responderlas con este lab.

---

## Parte J — Cleanup (opcional)

```
kubectl -n practice delete ingress practice-ingress
kubectl -n practice delete svc web-svc api-svc
kubectl -n practice delete deployment web-app api-app
kubectl delete namespace ingress-nginx
```

---
