## Workshop 11 — Services & Networking en Kubernetes

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **explicar, diseñar y depurar**:

- Por qué los **Pods no deben exponerse directamente**
- Qué es un **Service** y por qué existe
- Diferencias claras entre:
  - `ClusterIP`
  - `NodePort`
  - `LoadBalancer`
- Cómo funciona el **Service Discovery**
- DNS interno (`<service>.<namespace>.svc.cluster.local`)
- Flujo real de tráfico dentro del cluster
- Cómo responder **preguntas de entrevista** sobre networking en K8s

---

## Regla 0 — Contexto y orden (CRÍTICO)

Antes de ejecutar **cualquier comando**, valida:

```
kubectl config current-context
kubectl get nodes
```

Contexto esperado:

```
kind-k8s-lab
```

Si no estás ahí:

```
kubectl config use-context kind-k8s-lab
```

---

## Estado inicial esperado

- Cluster KIND activo
- Namespace `practice`
- Deployment `web-app` (nginx) con 2 réplicas

Verifica:

```
kubectl -n practice get deploy,pods
```

Resultado esperado:
- 2 pods `web-app-*` en estado `Running`

---

## Concepto clave (mentalidad Kubernetes)

> **Los Pods son efímeros  
> Los Services son estables**

Un Service **no expone Pods**, expone un **selector**.

---

## Arquitectura mental

```
Cliente
  |
Service (IP virtual estable)
  |
Selector (labels)
  |
Pods (IPs cambiantes)
```

---

## Paso 1 — ClusterIP (default y más usado)

### Qué es ClusterIP
- IP virtual **solo dentro del cluster**
- No accesible desde fuera
- Ideal para microservicios internos

---

### 1.1 Crear Service ClusterIP

Comando:

```
kubectl -n practice expose deployment web-app \
  --name web-svc \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

Verificar:

```
kubectl -n practice get svc
```

Salida esperada:
- `web-svc` con una IP tipo `10.x.x.x`

---

### 1.2 Inspeccionar Service

```
kubectl -n practice describe svc web-svc
```

Observa:
- Selector
- Endpoints
- Tipo de Service

---

## Paso 2 — Endpoints (lo que realmente importa)

### Qué son los Endpoints
- Lista real de IPs de Pods detrás del Service
- Se actualizan automáticamente

Ver:

```
kubectl -n practice get endpoints web-svc
```

Resultado:
- IPs de los pods `web-app`

---

## Paso 3 — DNS interno (Service Discovery)

### Regla DNS estándar

```
<service>.<namespace>.svc.cluster.local
```

Para este lab:

```
web-svc.practice.svc.cluster.local
```

---

### 3.1 Probar DNS desde un Pod

Crear pod temporal:

```
kubectl -n practice run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  --command -- sleep 3600
```

Entrar al pod:

```
kubectl -n practice exec -it dns-test -- sh
```

Dentro del pod:

```
nslookup web-svc
wget -qO- http://web-svc
```

Salir:

```
exit
```

---

## Paso 4 — NodePort (exposición básica externa)

### Qué es NodePort
- Abre un puerto en **cada nodo**
- Rango típico: `30000–32767`
- No recomendado en producción

---

### 4.1 Crear NodePort

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
- Puerto tipo `30xxx`

---

### 4.2 Acceder desde host

En KIND:

```
curl http://localhost:<NODEPORT>
```

---

## Paso 5 — LoadBalancer (concepto cloud)

### Qué es LoadBalancer
- Crea un LB del proveedor cloud
- No funciona nativamente en KIND
- En AKS/EKS/GKE → Azure LB / ELB / GLB

---

### 5.1 Crear LoadBalancer (solo demostrativo)

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

Resultado en KIND:
- EXTERNAL-IP `<pending>`

---

## Tabla definitiva de Services (memorizar)

| Tipo | Acceso | Uso real |
|----|------|--------|
| ClusterIP | Interno | Microservicios |
| NodePort | Básico externo | Labs |
| LoadBalancer | Externo | Producción |

---

## Flujo real de red (explicación de entrevista)

1. Cliente → Service IP
2. Service → kube-proxy
3. kube-proxy → Pod IP
4. Pod responde
5. Cliente no ve Pods

---

## Errores comunes (banderas rojas)

### ❌ Exponer Pods directamente
### ❌ Usar NodePort en producción
### ❌ No entender Endpoints
### ❌ Ignorar DNS interno

---

## Debugging rápido (runbook)

```
kubectl get svc -n practice
kubectl get endpoints -n practice
kubectl get pods -n practice -o wide
kubectl describe svc web-svc
```

---

## Cleanup (recomendado)

```
kubectl -n practice delete svc web-svc web-nodeport web-lb
kubectl -n practice delete pod dns-test
```

---
