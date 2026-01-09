## Workshop 7 — Recursos, Requests, Limits y QoS en Kubernetes

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **explicar y aplicar con seguridad**:

- Qué son **CPU y Memory** en Kubernetes
- Diferencia crítica entre:
  - `requests`
  - `limits`
- Qué es **QoS Class** y por qué importa en producción
- Por qué ocurren:
  - `OOMKilled`
  - throttling de CPU
- Cómo **diseñar recursos correctos** (ni sub ni over-provision)
- Cómo responder estas preguntas en **entrevistas técnicas**

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
- Sin recursos definidos (best-effort)

Verifica:

```
kubectl -n practice get deploy web-app -o yaml | grep resources -n
```

Si **NO aparece nada**, estás listo.

---

## Conceptos clave (ANTES de ejecutar comandos)

### CPU en Kubernetes
- Se mide en:
  - `1` = 1 core
  - `500m` = 0.5 core
- CPU **no se mata**, se **limita (throttle)**

### Memoria en Kubernetes
- Se mide en:
  - `Mi`, `Gi`
- Memoria **sí se mata** → `OOMKilled`

---

## Requests vs Limits (concepto central)

| Campo     | Qué significa | Impacto real |
|----------|---------------|--------------|
| request  | Lo mínimo garantizado | Scheduler decide |
| limit    | Máximo permitido | Kernel aplica |

Frase de entrevista:
> “Requests afectan scheduling; limits afectan runtime.”

---

## QoS Classes (clave enterprise)

| QoS | Condición |
|----|----------|
| Guaranteed | request = limit (CPU + MEM) |
| Burstable | request < limit |
| BestEffort | sin requests ni limits |

---

## Paso 1 — Ver estado actual (BestEffort)

### 1.1 Ver QoS actual

```
kubectl -n practice get pod -o jsonpath='{range .items[*]}{.metadata.name}{" → "}{.status.qosClass}{"\n"}{end}'
```

Resultado esperado:
```
BestEffort
```

---

## Paso 2 — Definir requests y limits (Burstable)

### 2.1 Editar Deployment

```
kubectl -n practice edit deployment web-app
```

Dentro del contenedor agrega:

```
resources:
  requests:
    cpu: "100m"
    memory: "128Mi"
  limits:
    cpu: "500m"
    memory: "256Mi"
```

Guarda y sal.

---

## Paso 3 — Observar el rollout

```
kubectl -n practice get pods -w
```

Pods antiguos mueren, nuevos nacen.

---

## Paso 4 — Ver QoS después del cambio

```
kubectl -n practice get pod -o jsonpath='{range .items[*]}{.metadata.name}{" → "}{.status.qosClass}{"\n"}{end}'
```

Resultado esperado:
```
Burstable
```

---

## Paso 5 — Forzar un OOMKilled (laboratorio controlado)

⚠️ **Este paso es didáctico**

### 5.1 Reducir límite de memoria

Edita el deployment:

```
limits:
  memory: "32Mi"
```

Aplica y observa:

```
kubectl -n practice get pods -w
```

Resultado esperado:
- Pod entra en `CrashLoopBackOff`

### 5.2 Confirmar causa

```
kubectl -n practice describe pod <pod>
```

Busca:
```
Reason: OOMKilled
```

---

## Paso 6 — Recuperar el sistema

Restablece valores seguros:

```
limits:
  memory: "256Mi"
```

Reinicia:

```
kubectl -n practice rollout restart deployment web-app
```

---

## Paso 7 — Guaranteed QoS (modo crítico)

### 7.1 Igualar request y limit

```
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "250m"
    memory: "256Mi"
```

### 7.2 Ver QoS

```
kubectl -n practice get pod -o jsonpath='{range .items[*]}{.metadata.name}{" → "}{.status.qosClass}{"\n"}{end}'
```

Resultado esperado:
```
Guaranteed
```

---

## Comparación práctica (entrevista)

| Caso | QoS | Uso típico |
|----|----|----|
| BestEffort | Labs, pruebas | ❌ No producción |
| Burstable | Apps web | ✅ Recomendado |
| Guaranteed | DB, pagos | ⚠️ Crítico |

---

## Errores comunes (y cómo explicarlos)

### E1) “Mi pod se cae sin logs”
Causa:
- OOMKilled
Solución:
- Aumentar memory limit
- Analizar consumo real

---

### E2) “Mi app va lenta”
Causa:
- CPU throttling
Solución:
- Ajustar cpu limits
- Observar métricas

---

## Cleanup del workshop

(Opcional)

```
kubectl delete namespace practice --ignore-not-found=true
```

---

## Guion de entrevista (listo)

- “Defino requests para scheduling y limits para runtime.”
- “Evito BestEffort en producción.”
- “Uso Burstable para la mayoría de workloads.”
- “Guaranteed solo para servicios críticos.”
- “OOMKilled es señal de memory limit mal definido.”

---
