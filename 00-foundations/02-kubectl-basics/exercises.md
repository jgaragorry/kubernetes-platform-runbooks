# 00-foundations/02-kubectl-basics/exercises.md

## Objetivo del workshop (nivel “ojos cerrados”)

Al finalizar este módulo vas a poder ejecutar, explicar y diagnosticar **sin perderte**:

- Contextos (`kubectl config ...`) y por qué son críticos (evitar operar en el cluster equivocado)
- Namespaces como unidad de trabajo/aislamiento
- Diferencia real entre:
  - **Pod efímero** (sin controlador)
  - **Deployment** (estado deseado + autocuración)
  - **ReplicaSet** (mecanismo que mantiene las réplicas)
  - **Service ClusterIP** (exposición interna estable)
- Lectura de estado con `get -o wide`, `describe`, `events`
- Troubleshooting corto: “¿por qué no existe el namespace?” y “¿por qué no conecta al API?”

---

## Reglas de este módulo (para no desorientarte)

### Regla 0 — Un cluster a la vez
Si tienes **kind** + **EKS/AKS** en la misma VM, **no improvises**.

Siempre comienza con:

```
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Si alguno falla: **STOP**.

### Regla 1 — Un namespace por ejercicio
En este módulo usaremos:

- Namespace: `practice`

### Regla 2 — No mezclar sintaxis de recursos
Forma correcta:

- `kubectl delete pod/<POD_NAME>`
- `kubectl get deploy,rs,pods`

Forma incorrecta (tu error real):

- `kubectl delete pod pod/<POD_NAME>`

---

## Pre-check (antes de cualquier ejercicio)

> Objetivo: garantizar que estás apuntando al cluster correcto y que el control plane responde.

### 1) Verifica el contexto actual

```
kubectl config current-context
```

**Resultado esperado (ejemplo kind):**
- `kind-k8s-lab` (o el nombre de tu cluster kind)

**Si aparece EKS/AKS:**
- Ver sección “Troubleshooting: me fui a EKS/AKS sin querer”

### 2) Verifica conectividad con el API Server

```
kubectl cluster-info
```

**Resultado esperado (kind):**
- “Kubernetes control plane is running at https://127.0.0.1:xxxxx”

### 3) Verifica nodos listos

```
kubectl get nodes
```

**Resultado esperado:**
- Al menos 1 nodo
- `STATUS = Ready`

### 4) Verifica pods del sistema

```
kubectl get pods -n kube-system
```

**Resultado esperado:**
- `coredns` Running
- `kube-proxy` Running
- componentes control-plane Running

---

## Ejercicio 1 — Namespace “practice” (unidad de trabajo)

> Objetivo: crear un namespace y comprobarlo.

### 1.1 Crear el namespace

```
kubectl create namespace practice
```

**Resultado esperado:**
- `namespace/practice created`

**Si ya existe:**
- verás: `AlreadyExists` (esto es OK)

### 1.2 Confirmar que existe

```
kubectl get ns | grep practice || true
```

**Resultado esperado:**
- una línea con `practice   Active   ...`

---

## Ejercicio 2 — Pod efímero (sin autocuración)

> Objetivo: entender por qué un Pod directo **no** es “estado deseado”.

### 2.1 Crear un pod “directo”

```
kubectl -n practice run test-pod --image=nginx:1.27 --restart=Never
```

**Qué significa cada cosa:**
- `-n practice`: crea el recurso en el namespace practice
- `run test-pod`: nombre del pod
- `--image=nginx:1.27`: imagen del contenedor
- `--restart=Never`: fuerza que sea Pod directo (no Deployment)

### 2.2 Observar estado + IP + nodo

```
kubectl -n practice get pods -o wide
```

**Resultado esperado:**
- `STATUS = Running`
- IP `10.244.x.y` (en kind suele ser así)
- NODE = tu nodo kind

### 2.3 Eliminar el pod

```
kubectl -n practice delete pod test-pod
```

**Resultado esperado:**
- `pod "test-pod" deleted`

### 2.4 Confirmar que NO vuelve

```
kubectl -n practice get pods
```

**Resultado esperado:**
- `No resources found...`

**Interpretación clave:**
- No hay controlador manteniendo estado deseado → no hay autocuración.

**Frase entrevista (corta):**
- “Un pod directo es para pruebas. Producción usa controladores como Deployment.”

---

## Ejercicio 3 — Deployment (estado deseado + autocuración)

> Objetivo: crear un Deployment y ver la relación Deployment → ReplicaSet → Pods.

### 3.1 Crear deployment

```
kubectl -n practice create deployment web-app --image=nginx:1.27
```

**Resultado esperado:**
- `deployment.apps/web-app created`

### 3.2 Escalar a 2 réplicas

```
kubectl -n practice scale deployment web-app --replicas=2
```

**Resultado esperado:**
- `deployment.apps/web-app scaled`

### 3.3 Ver jerarquía completa

```
kubectl -n practice get deploy,rs,pods -o wide
```

**Qué debes observar:**
- Deployment `web-app` con `READY 2/2`
- ReplicaSet con `DESIRED 2`
- 2 pods `web-app-<hash>-<suffix>`

### 3.4 Probar autocuración: borrar 1 pod

1) Lista pods:

```
kubectl -n practice get pods
```

2) Elige uno y bórralo (forma correcta):

```
kubectl -n practice delete pod/<POD_NAME>
```

Ejemplo (no copies el nombre, usa el tuyo):

```
kubectl -n practice delete pod/web-app-7fd7f747cd-zt9kt
```

**Resultado esperado:**
- `pod "<name>" deleted`

### 3.5 Observar recreación en vivo

```
kubectl -n practice get pods -w
```

**Resultado esperado:**
- aparece un nuevo pod con nombre distinto
- vuelve a haber 2 pods en Running

**Interpretación clave:**
- El Deployment mantiene estado deseado
- El ReplicaSet converge a ese estado deseado

**Frase entrevista:**
- “Deployment aplica reconciliación: si muere un pod, ReplicaSet crea otro para mantener replicas.”

---

## Ejercicio 4 — Describe (estado + estrategia + eventos)

> Objetivo: aprender a leer “qué pasó” sin adivinar.

### 4.1 Describe del deployment

```
kubectl -n practice describe deployment web-app
```

**Qué buscar (checklist):**
- `Replicas: 2 desired | 2 updated | 2 total | 2 available`
- StrategyType: RollingUpdate
- Events: “Scaled up replica set…”

### 4.2 Describe del replicaset (RS)

1) Obtén el nombre del RS:

```
kubectl -n practice get rs
```

2) Describe:

```
kubectl -n practice describe rs <RS_NAME>
```

**Qué buscar:**
- Labels / selector (`app=web-app`, `pod-template-hash=...`)
- Events: Created pod …

**Interpretación clave:**
- Si el selector/labels no matchean, Services no van a enrutar.

---

## Ejercicio 5 — Service ClusterIP (exposición interna estable)

> Objetivo: crear un Service y entender `port` vs `targetPort`.

### 5.1 Crear el Service (ClusterIP)

```
kubectl -n practice expose deployment web-app \
  --name web-svc \
  --port 80 \
  --target-port 80 \
  --type ClusterIP
```

**Qué significa:**
- `expose deployment web-app`: crea un Service apuntando al selector del deployment
- `--name web-svc`: nombre del Service
- `--port 80`: puerto del Service (lo que el cliente usa dentro del cluster)
- `--target-port 80`: puerto del contenedor/pod (donde escucha nginx)
- `--type ClusterIP`: solo accesible dentro del cluster

### 5.2 Ver el service

```
kubectl -n practice get svc -o wide
```

**Resultado esperado:**
- `web-svc` con `TYPE ClusterIP`
- `CLUSTER-IP` asignada
- `PORT(S) 80/TCP`

### 5.3 Ver endpoints (confirmación real de ruteo)

```
kubectl -n practice get endpoints web-svc -o wide
```

**Resultado esperado:**
- 2 endpoints (IPs de los pods) con `:80`

**Interpretación clave:**
- Service *sin endpoints* = selector no matchea o pods no Ready.

**Frase entrevista:**
- “Valido service con endpoints; si están vacíos, es un problema de labels/selector o readiness.”

---

## Ejercicio 6 — Cleanup (mantener el namespace limpio)

> Objetivo: practicar limpieza idempotente y dejar el namespace listo.

### 6.1 Eliminar service y deployment

```
kubectl -n practice delete svc web-svc --ignore-not-found=true
kubectl -n practice delete deployment web-app --ignore-not-found=true
```

**Qué hace `--ignore-not-found=true`:**
- Hace el comando idempotente: si no existe, no falla.

### 6.2 Confirmar que no queda nada

```
kubectl -n practice get all
```

**Resultado esperado:**
- “No resources found” o vacío

---

## Checklist de “resultado esperado” del módulo

Si todo salió bien, debes poder demostrar:

1) `kubectl config current-context` muestra tu cluster kind  
2) `kubectl get ns | grep practice` devuelve Active  
3) Puedes crear y borrar un pod directo (no vuelve)  
4) Puedes crear un deployment y borrar un pod (vuelve)  
5) Puedes crear un service ClusterIP y ver endpoints poblados  
6) Puedes limpiar todo con delete idempotente

---

## Troubleshooting (lo más importante del documento)

### A) “Error: namespaces "practice" not found” (pero yo lo creé…)

**Causa #1 (la más común, y te pasó):** cambiaste de contexto (EKS/AKS)  
**Cómo detectarlo:**

```
kubectl config current-context
kubectl cluster-info
```

Si ves un endpoint EKS/AKS, estás en otro cluster.

**Solución (ejemplo kind):**

1) Lista contextos:

```
kubectl config get-contexts
```

2) Cambia al de kind:

```
kubectl config use-context kind-k8s-lab
```

3) Confirma:

```
kubectl config current-context
kubectl get ns | grep practice || true
```

---

### B) “Unable to connect to the server: ... eks.amazonaws.com ... no such host”

Eso no es “kubernetes roto”. Es:

- Estás apuntando a EKS/AKS
- Tu VM no resuelve o no tiene ruta a ese endpoint
- O tu kubeconfig apunta a un cluster que no es accesible desde esa red

**Solución:**
Vuelve a kind y valida:

```
kubectl config use-context kind-k8s-lab
kubectl cluster-info
kubectl get nodes
```

---

### C) Me falla `expose deployment ...` con `AlreadyExists`

Eso es normal si ya existe el service. Solución idempotente:

```
kubectl -n practice delete svc web-svc --ignore-not-found=true
kubectl -n practice expose deployment web-app \
  --name web-svc --port 80 --target-port 80 --type ClusterIP
```

---

### D) “Service no enruta” / endpoints vacíos

Diagnóstico en orden:

1) ¿existe el service?

```
kubectl -n practice get svc web-svc
```

2) ¿endpoints?

```
kubectl -n practice get endpoints web-svc -o wide
```

3) ¿pods Ready?

```
kubectl -n practice get pods -o wide
```

4) ¿labels del pod y selector del service?

```
kubectl -n practice describe svc web-svc
kubectl -n practice get pods --show-labels
```

---

## Notas para entrevistas (lo que NO puede faltar)

- “Antes de operar, verifico contexto y salud del cluster.”
- “Un Pod directo es útil para pruebas. Para producción uso Deployment.”
- “Deployment mantiene desired state, ReplicaSet implementa la reconciliación.”
- “Service ofrece endpoint estable; valido ruteo con endpoints, no con fe.”

---
