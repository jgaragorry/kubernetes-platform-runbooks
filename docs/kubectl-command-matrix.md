# Kubectl Command Matrix — Order, Meaning, Flags, Standards vs Customizable

## Objetivo

Este documento es tu **matriz de comandos kubectl** (nivel runbook enterprise):

- Qué hace cada comando
- Por qué se ejecuta en ese orden
- Qué flags/argumentos son estándar vs personalizables
- Errores típicos (incluyendo el caso real que te pasó: **cambio de contexto a EKS**)
- Qué debes poder explicar en entrevistas

---

## Convención de lectura (muy importante)

En Kubernetes, casi todo se entiende con esta secuencia mental:

1) **Contexto / Cluster** (¿a qué cluster estoy apuntando?)  
2) **Namespace** (¿dónde estoy trabajando?)  
3) **Recurso** (¿qué creé / consulté?)  
4) **Estado deseado vs estado actual**  
5) **Eventos** (¿qué ocurrió?)  
6) **Logs** (¿qué dice el contenedor?)  

Si te saltas 1) o 2), te pierdes (exactamente lo que te pasó).

---

## Matriz de comandos (tabla)

> Nota: “Orden recomendado” no significa “obligatorio siempre”, significa:  
> el orden que **minimiza errores** y te mantiene orientado en labs/entrevistas.

| # | Comando | ¿Se ejecuta en este orden? | ¿Qué hace? | Flags/argumentos (qué significan) | Estándar vs Personalizable | Errores típicos / Anti-patrones | Qué decir en entrevista |
|---|---------|----------------------------|------------|-----------------------------------|----------------------------|----------------------------------|-------------------------|
| 1 | `kubectl config get-contexts` | Sí, al inicio | Lista todos los contextos disponibles en tu kubeconfig | Sin flags | Estándar | No revisarlo y operar en el cluster equivocado | “Siempre valido mis contextos antes de tocar recursos.” |
| 2 | `kubectl config current-context` | Sí, al inicio | Muestra el contexto activo | Sin flags | Estándar | Confiarte “de memoria” | “Verifico contexto activo para evitar cambios en prod.” |
| 3 | `kubectl config use-context <ctx>` | Sí, cuando cambias de cluster | Cambia el cluster/contexto activo | `<ctx>` = nombre del contexto | Estándar (pero `<ctx>` es variable) | Tener EKS + kind y terminar en EKS por accidente | “Cambio explícito y verifico con current-context.” |
| 4 | `kubectl cluster-info` | Sí, después de confirmar contexto | Confirma API server y servicios base (DNS proxy) | Sin flags | Estándar | Saltarlo y asumir que el cluster está ok | “Valido conectividad con el control plane.” |
| 5 | `kubectl get nodes` | Sí, temprano | Verifica que el/los nodos están `Ready` | Sin flags | Estándar | No revisar `STATUS` y culpar a kubectl/app | “Si nodos no están Ready, paro y diagnostico.” |
| 6 | `kubectl get pods -n kube-system` | Sí, temprano | Verifica componentes core (coredns, kube-proxy, etc.) | `-n` namespace | Estándar (namespace fijo aquí) | Ignorar `kube-system` y perseguir fallos falsos | “Siempre reviso salud del plano base antes de apps.” |
| 7 | `kubectl get ns` | Sí, antes de crear cosas | Lista namespaces existentes | Sin flags | Estándar | Crear recursos en `default` por accidente | “Uso namespaces para aislamiento y claridad.” |
| 8 | `kubectl create namespace <ns>` | Sí, antes de desplegar apps | Crea un namespace nuevo | `<ns>` = nombre | Estándar (pero `<ns>` personalizable) | Olvidar crearlo y luego `NotFound` | “Creo namespaces explícitos por entorno/lab.” |
| 9 | `kubectl -n <ns> run <pod> --image=<img> --restart=Never` | Sí, como primer ejercicio | Crea un **Pod directo** (efímero; no autocura) | `-n` ns, `--image` imagen, `--restart=Never` evita Deployment | Semi-estándar (para learning) | Usarlo como producción | “Pod directo es útil para testing, no para HA.” |
| 10 | `kubectl -n <ns> get pods -o wide` | Sí, inmediato después de crear | Verifica estado, IP, nodo | `-o wide` más columnas | Estándar | No mirar `STATUS` y seguir creando cosas | “Observo estado y scheduling (IP, Node).” |
| 11 | `kubectl -n <ns> delete pod <pod>` | Sí, para demostrar efímero | Borra el pod (no se recrea si fue creado con `run --restart=Never`) | `-n` ns | Estándar | Creer que “k8s autocura todo” | “Autocuración depende del controlador.” |
| 12 | `kubectl -n <ns> create deployment <name> --image=<img>` | Sí, base real | Crea Deployment (controlador) | `deployment` + nombre + `--image` | Estándar | Crear deployments sin namespace claro | “Deployment define estado deseado y permite rolling updates.” |
| 13 | `kubectl -n <ns> scale deployment <name> --replicas=<n>` | Sí, después del deployment | Ajusta réplicas (estado deseado) | `--replicas` número | Estándar (valor personalizable) | Escalar sin revisar recursos/limits | “Cambio estado deseado; controller converge.” |
| 14 | `kubectl -n <ns> get deploy,rs,pods` | Sí, para ver relación | Muestra jerarquía: Deployment → ReplicaSet → Pods | lista múltiple de recursos | Estándar | No entender qué crea qué | “Deployment gestiona RS; RS gestiona Pods.” |
| 15 | `kubectl -n <ns> delete pod/<pod>` | Sí, para demostrar autocura | Borra un pod gestionado por RS; **se recrea** | `pod/<name>` forma corta | Estándar | Usar sintaxis incorrecta: `delete pod pod/<name>` | “Controller recrea pods para mantener desired state.” |
| 16 | `kubectl -n <ns> get pods -w` | Sí, para observar | Watch: ver recreación en tiempo real | `-w` watch | Estándar | Olvidar salir y confundir la salida | “Uso watch para validar reconciliación.” |
| 17 | `kubectl -n <ns> describe deployment <name>` | Sí, cuando necesitas detalle | Muestra condiciones, estrategia, eventos | `describe` | Estándar | Confundir `describe` con `logs` | “Describe me da eventos/estado del controlador.” |
| 18 | `kubectl -n <ns> describe rs <name>` | Sí, para entender controlador | Muestra selector/labels, eventos, pods | `rs` replicaset | Estándar | No revisar selector/labels | “Si selector está mal, Service no enruta.” |
| 19 | `kubectl -n <ns> expose deployment <name> --name <svc> --port 80 --target-port 80 --type ClusterIP` | Sí, después de deployment | Crea Service para exponer pods internamente | `--port` servicio, `--target-port` contenedor, `--type` | Estándar (valores personalizables) | Error: namespace no existe / contexto incorrecto | “Service abstrae pods; mantiene endpoint estable.” |
| 20 | `kubectl -n <ns> get svc` | Sí, luego del expose | Verifica ClusterIP, puertos | `get svc` | Estándar | Asumir acceso externo con ClusterIP | “ClusterIP es interno; para fuera uso NodePort/Ingress/LB.” |
| 21 | `kubectl -n <ns> describe svc <svc>` | Sí, si no enruta | Muestra endpoints, selector, puertos | `describe` | Estándar | No revisar endpoints (si están vacíos, selector no matchea) | “Si endpoints vacío, es un problema de labels/selector.” |
| 22 | `kubectl -n <ns> get endpoints <svc>` | Sí, troubleshooting | Verifica endpoints reales detrás del Service | `endpoints` | Estándar | Ignorarlo y culpar a networking | “Endpoints confirman si el service tiene backends.” |
| 23 | `kubectl -n <ns> logs <pod>` | Sí, troubleshooting app | Logs del contenedor | `-f` follow, `--tail` | Estándar (parámetros personalizables) | Hacer logs sin confirmar pod correcto | “Logs para app-level; describe para controller-level.” |
| 24 | `kubectl -n <ns> exec -it <pod> -- <cmd>` | Sí, debugging controlado | Ejecuta comando dentro del contenedor | `-it` interactivo, `--` fin de flags | Estándar | Usarlo como “solución permanente” | “Exec es diagnóstico; la solución debe ser declarativa.” |

---

## Lo que te pasó (caso real): “namespace practice not found” + “Unable to connect… EKS … DNS”

### Síntoma que viste

- `Error from server (NotFound): namespaces "practice" not found`
- Luego: `Unable to connect to the server: ... eks.amazonaws.com ... no such host`
- Y después viste que tu `current-context` era algo tipo EKS (`arn:aws:eks:...`)

### Causa raíz (Root Cause)

No fue Kubernetes “olvidando” el namespace.

Fue:

1) Te moviste a otro contexto (EKS) sin darte cuenta  
2) Estabas ejecutando comandos contra otro cluster  
3) En ese cluster:
   - o no existía `practice`
   - o no había conectividad/DNS desde esa VM a ese endpoint
4) Por eso se mezclaron errores (NotFound + no host)

### Prevención (regla oro)

Antes de cualquier comando “operativo”, ejecuta siempre:

```
kubectl config current-context
kubectl cluster-info
kubectl get nodes
```

Si cualquiera falla, **no continúes**.

---

## Estándar vs Personalizable (reglas claras)

### Estándar (no se negocia en este repositorio)

- Validar contexto antes de operar
- Trabajar por namespace (`-n <ns>`)
- Usar `deployment` para apps persistentes
- Usar `describe` + `events` para diagnosticar controlador
- Diferenciar:
  - Pod efímero (sin controller)
  - Pod bajo controller (Deployment/ReplicaSet)

### Personalizable (depende del ejercicio)

- Nombre de namespace (`labs`, `practice`, etc.)
- Nombre de recursos (`web-app`, `web-svc`, etc.)
- Imagen (`nginx:1.27`, `httpd`, etc.)
- Cantidad de réplicas (`--replicas=2`, `3`, `5`)
- Tipo de Service (`ClusterIP`, `NodePort`, `LoadBalancer`)

---

## Ejercicio mínimo (repetible) con orden “anti-desorientación”

Este es el flujo corto que debes repetir hasta hacerlo “con ojos cerrados”:

1) `kubectl config current-context`
2) `kubectl cluster-info`
3) `kubectl get nodes`
4) `kubectl create ns practice` (una sola vez)
5) `kubectl -n practice create deployment web-app --image=nginx:1.27`
6) `kubectl -n practice scale deployment web-app --replicas=2`
7) `kubectl -n practice get deploy,rs,pods -o wide`
8) `kubectl -n practice expose deployment web-app --name web-svc --port 80 --target-port 80 --type ClusterIP`
9) `kubectl -n practice get svc,ep`
10) `kubectl -n practice delete pod/<uno>` (ver autocura)
11) `kubectl -n practice get pods -w` (ver recreación)

---

## Checklist “si algo falla”

Si aparece cualquier error raro, ejecuta en este orden:

1) `kubectl config current-context`
2) `kubectl cluster-info`
3) `kubectl get nodes`
4) `kubectl get ns | grep <ns> || true`
5) `kubectl -n <ns> get deploy,rs,pods,svc,ep`

No adivines. Observa.

---

