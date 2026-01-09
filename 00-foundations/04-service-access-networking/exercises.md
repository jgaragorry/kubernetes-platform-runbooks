## Workshop 5 — Acceso a Servicios en Kubernetes (Networking Básico)

### Objetivo (lo que vas a dominar)
Al finalizar este workshop vas a poder **explicar y demostrar**:

- Cómo se accede a una app en Kubernetes **SIN exponerla a Internet**
- Diferencia real entre:
  - Pod
  - Service (ClusterIP)
  - Acceso interno vs externo
- Uso correcto de:
  - `kubectl port-forward`
  - `kubectl exec`
  - Pod cliente (busybox / curl)
- Cómo **probar networking como lo haría un SRE**
- Errores clásicos de entrevistas y cómo evitarlos

---

## Reglas del módulo

### Regla 0 — Contexto y namespace
Antes de empezar:

```
kubectl config current-context
kubectl get nodes
kubectl get ns | grep practice
```

Si `practice` no existe o el contexto no es el correcto → **NO SIGAS**.

---

## Estado inicial esperado

Este workshop asume que **YA EXISTE**:

- Namespace: `practice`
- Deployment: `web-app`
- Service: `web-svc` (ClusterIP)
- Pods nginx corriendo

Verifícalo:

```
kubectl -n practice get deploy,svc,pods -o wide
kubectl -n practice get endpoints web-svc -o wide
```

Si los endpoints están vacíos → vuelve al Workshop 4.

---

## Paso 1 — Acceso directo a un Pod (NO recomendado en producción)

### 1.1 Obtener IP del pod

```
kubectl -n practice get pods -o wide
```

Ejemplo:

```
web-app-xxxxx   Running   10.244.0.12
```

### 1.2 Probar acceso desde el **nodo** (solo funciona en kind)

```
curl http://10.244.0.12
```

**Qué estás aprendiendo aquí**
- Cada Pod tiene IP propia
- Esa IP es **efímera**
- Si el pod muere, esa IP desaparece

**Frase de entrevista**
> “Nunca expongo apps por IP de pod porque no es estable.”

---

## Paso 2 — Acceso mediante Service (ClusterIP) — el camino correcto

### 2.1 Ver IP del Service

```
kubectl -n practice get svc web-svc
```

Ejemplo:

```
web-svc   ClusterIP   10.96.123.45
```

### 2.2 Intentar acceder desde el host (fallará)

```
curl http://10.96.123.45
```

**Resultado esperado**
- ❌ No responde

**Por qué**
- `ClusterIP` es **solo accesible dentro del cluster**

---

## Paso 3 — Acceso interno real (SRE style)

### 3.1 Crear un pod cliente temporal

```
kubectl -n practice run net-test \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
```

Verifica:

```
kubectl -n practice get pods
```

### 3.2 Entrar al pod cliente

```
kubectl -n practice exec -it net-test -- sh
```

### 3.3 Probar Service desde dentro del cluster

```
wget -qO- http://web-svc
```

O usando DNS completo:

```
wget -qO- http://web-svc.practice.svc.cluster.local
```

**Qué aprendiste**
- Kubernetes provee DNS interno
- Services son el punto estable de acceso
- Los pods nunca se comunican por IP directa

Salir del pod:

```
exit
```

---

## Paso 4 — Port-forward (acceso local sin exponer)

### 4.1 Port-forward al Service

```
kubectl -n practice port-forward svc/web-svc 8080:80
```

En otra terminal:

```
curl http://localhost:8080
```

**Qué está pasando**
- Tu máquina local → túnel kubectl → Service → Pod
- Ideal para debugging, demos, workshops
- ❌ No es producción

**Frase de entrevista**
> “Uso port-forward solo para troubleshooting o desarrollo local.”

---

## Paso 5 — Port-forward directo a un Pod (para debugging)

### 5.1 Elegir un pod

```
kubectl -n practice get pods
```

### 5.2 Port-forward al pod

```
kubectl -n practice port-forward pod/<POD_NAME> 8081:80
```

Probar:

```
curl http://localhost:8081
```

**Diferencia clave**
- Service → balancea
- Pod → apunta a una instancia específica

---

## Paso 6 — Comparación conceptual (CLAVE para entrevistas)

| Método | Estable | Balancea | Uso real |
|------|--------|----------|---------|
| IP de Pod | ❌ | ❌ | Nunca |
| Service (ClusterIP) | ✅ | ✅ | Comunicación interna |
| Port-forward | ⚠️ | ❌ | Debug / Dev |
| NodePort | ⚠️ | ✅ | Labs / legacy |
| Ingress | ✅ | ✅ | Producción |

---

## Errores típicos (y cómo detectarlos)

### E1) “No puedo acceder al Service”
Checklist rápido:

```
kubectl -n practice get svc
kubectl -n practice get endpoints web-svc
kubectl -n practice get pods --show-labels
```

Causa típica:
- selector del Service ≠ labels del Pod

---

### E2) “Funciona por pod pero no por service”
Diagnóstico:
- Service no encuentra endpoints
- Error de labels

---

### E3) “kubectl no conecta al cluster”
Diagnóstico inmediato:

```
kubectl config current-context
kubectl cluster-info
```

Solución:
```
kubectl config use-context kind-k8s-lab
```

---

## Cleanup del workshop

```
kubectl -n practice delete pod net-test --ignore-not-found=true
```

(O borra todo el namespace si vas a repetir)

```
kubectl delete namespace practice --ignore-not-found=true
```

---

## Guion de entrevista (resumen corto y potente)

- “Los pods son efímeros, los services son estables.”
- “ClusterIP es acceso interno, no expuesto.”
- “Uso DNS interno, no IPs.”
- “Port-forward solo para debugging.”
- “Siempre valido endpoints, no solo servicios.”

---

