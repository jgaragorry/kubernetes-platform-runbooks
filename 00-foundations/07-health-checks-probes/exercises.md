## Workshop 8 — Health Checks en Kubernetes (liveness, readiness, startup)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **diseñar y explicar**:

- Diferencia real entre:
  - `livenessProbe`
  - `readinessProbe`
  - `startupProbe`
- Por qué **NO** son lo mismo
- Cómo evitar:
  - reinicios infinitos
  - tráfico a pods no listos
  - downtime en despliegues
- Cómo responder **preguntas de entrevista senior** sobre probes

---

## Reglas del módulo

### Regla 0 — Contexto correcto
Antes de continuar:

```
kubectl config current-context
kubectl get nodes
```

Debes estar en:
```
kind-k8s-lab
```

---

## Estado inicial esperado

Este workshop asume:

- Namespace: `practice`
- Deployment: `web-app`
- Imagen: `nginx:1.27`
- Réplicas: 2
- **Sin probes configuradas**

Verifica:

```
kubectl -n practice get deploy web-app -o yaml | grep probe -n
```

Si **no aparece nada**, estás listo.

---

## Conceptos clave (ANTES de tocar YAML)

### Qué es un Probe
Un **probe** es un chequeo que Kubernetes ejecuta **dentro del contenedor** para decidir:

- ¿Lo reinicio?
- ¿Le mando tráfico?
- ¿Sigo esperando que arranque?

---

## Tipos de probes (visión mental correcta)

### 1️⃣ livenessProbe
Pregunta:
> “¿Este contenedor sigue vivo?”

Si falla:
- Kubernetes **REINICIA** el contenedor

Uso típico:
- Deadlocks
- App colgada

---

### 2️⃣ readinessProbe
Pregunta:
> “¿Este contenedor está listo para recibir tráfico?”

Si falla:
- Kubernetes **NO ENVÍA TRÁFICO**
- **NO reinicia** el pod

Uso típico:
- Warm-up
- Dependencias externas

---

### 3️⃣ startupProbe
Pregunta:
> “¿Ya terminó de arrancar?”

Mientras exista:
- liveness y readiness **NO se ejecutan**

Uso típico:
- Apps lentas
- JVM
- Migraciones iniciales

---

## Regla de oro (entrevista)
> “Startup protege el arranque, readiness protege el tráfico, liveness protege la salud.”

---

## Paso 1 — Ver comportamiento SIN probes

### 1.1 Simular fallo manual

```
kubectl -n practice exec deploy/web-app -- pkill nginx
```

Observa:

```
kubectl -n practice get pods -w
```

Resultado esperado:
- El pod **NO se reinicia**
- El servicio queda inconsistente

👉 Kubernetes **no sabe** que algo anda mal.

---

## Paso 2 — Agregar livenessProbe

### 2.1 Editar Deployment

```
kubectl -n practice edit deployment web-app
```

Agrega al contenedor:

```
livenessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
```

Guarda y sal.

---

### 2.2 Observar rollout

```
kubectl -n practice get pods -w
```

---

### 2.3 Probar liveness

```
kubectl -n practice exec deploy/web-app -- pkill nginx
```

Resultado esperado:
- Pod entra en `Restarting`
- Kubernetes **recupera solo**

---

## Paso 3 — Agregar readinessProbe

### 3.1 Editar Deployment

Agrega (además del liveness):

```
readinessProbe:
  httpGet:
    path: /
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2
```

---

### 3.2 Observar endpoints

```
kubectl -n practice get endpoints
```

Luego rompe nginx:

```
kubectl -n practice exec deploy/web-app -- pkill nginx
```

Resultado esperado:
- Pod sigue vivo
- **SALE del endpoint**
- No recibe tráfico

---

## Paso 4 — Entender startupProbe (clave senior)

### Problema real
Apps lentas + liveness:
- Kubernetes mata la app **antes de arrancar**

---

### 4.1 Agregar startupProbe

```
startupProbe:
  httpGet:
    path: /
    port: 80
  failureThreshold: 30
  periodSeconds: 5
```

👉 Kubernetes esperará **hasta 150s** antes de juzgar.

---

## Paso 5 — Validar probes activas

```
kubectl -n practice describe pod <pod>
```

Busca secciones:
- Liveness
- Readiness
- Startup

---

## Tabla resumen (memorizable)

| Probe | Reinicia | Quita tráfico | Uso |
|----|----|----|----|
| liveness | ✅ | ❌ | App colgada |
| readiness | ❌ | ✅ | Warm-up |
| startup | ❌ | ❌ | Arranque lento |

---

## Errores comunes (pregunta clásica)

### ❌ “Uso solo liveness”
Problema:
- Reinicios innecesarios

### ❌ “Uso readiness para todo”
Problema:
- Nunca se recupera solo

### ❌ “No uso probes”
Problema:
- Kubernetes no puede ayudarte

---

## Frases de entrevista (golden answers)

- “Liveness reinicia, readiness controla tráfico.”
- “Startup protege el arranque.”
- “Nunca uso liveness sin pensar en startup.”
- “Readiness es clave para zero-downtime.”

---

## Cleanup (opcional)

```
kubectl delete namespace practice --ignore-not-found=true
```

---
