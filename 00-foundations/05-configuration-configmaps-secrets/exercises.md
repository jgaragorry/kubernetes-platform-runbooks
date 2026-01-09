## Workshop 6 — Configuración Externa en Kubernetes (ConfigMaps & Secrets)

### Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **diseñar, aplicar y explicar**:

- Diferencia real entre **imagen** y **configuración**
- Uso correcto de:
  - ConfigMap (datos no sensibles)
  - Secret (datos sensibles)
- Inyección de configuración vía:
  - Variables de entorno
  - Volúmenes
- Actualización de configuración **sin rebuild de imagen**
- Patrones enterprise:
  - separación config/app
  - rotación de secretos
  - idempotencia
- Errores comunes en entrevistas y producción

---

## Reglas del módulo

### Regla 0 — Contexto y namespace
Antes de empezar:

```
kubectl config current-context
kubectl get nodes
kubectl get ns | grep practice
```

Si `practice` no existe → créalo o vuelve al Workshop 4.

---

## Estado inicial esperado

Este workshop asume:

- Namespace: `practice`
- Deployment: `web-app` (nginx)
- Service: `web-svc`
- Pods Running

Verifica:

```
kubectl -n practice get deploy,svc,pods
```

---

## Conceptos clave (ANTES de tocar comandos)

### Imagen ≠ Configuración
- La **imagen** debe ser inmutable
- La **configuración** cambia por ambiente

### ConfigMap
- Texto plano
- No cifrado
- Ideal para:
  - flags
  - URLs
  - configs de app

### Secret
- Base64 (no cifrado por defecto)
- Debe tratarse como sensible
- Ideal para:
  - passwords
  - tokens
  - API keys

**Frase de entrevista**
> “Nunca bakeo configuración sensible dentro de la imagen.”

---

## Paso 1 — Crear un ConfigMap (imperativo → para entender)

### 1.1 Crear ConfigMap simple

```
kubectl -n practice create configmap web-config \
  --from-literal=APP_ENV=dev \
  --from-literal=APP_NAME=nginx-demo
```

Verifica:

```
kubectl -n practice get configmap web-config -o yaml
```

---

## Paso 2 — Crear un Secret (imperativo)

### 2.1 Crear Secret genérico

```
kubectl -n practice create secret generic web-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=supersecret
```

Verifica:

```
kubectl -n practice get secret web-secret -o yaml
```

Observa:
- valores en base64
- NO es cifrado real

---

## Paso 3 — Inyectar ConfigMap y Secret como variables de entorno

### 3.1 Editar Deployment (concepto)

Edita `deployment.yaml` o usa:

```
kubectl -n practice edit deployment web-app
```

Dentro del contenedor agrega:

```
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: web-config
        key: APP_ENV
  - name: APP_NAME
    valueFrom:
      configMapKeyRef:
        name: web-config
        key: APP_NAME
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: web-secret
        key: DB_USER
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: web-secret
        key: DB_PASSWORD
```

Guarda y sal.

### 3.2 Ver rollout

```
kubectl -n practice get pods -w
```

Esperado:
- Pods se recrean automáticamente

---

## Paso 4 — Validar variables dentro del pod

### 4.1 Entrar a un pod

```
kubectl -n practice exec -it deploy/web-app -- env | grep APP_
```

Resultado esperado:

```
APP_ENV=dev
APP_NAME=nginx-demo
```

### 4.2 Ver secret (sin imprimirlo)

```
kubectl -n practice exec -it deploy/web-app -- env | grep DB_
```

---

## Paso 5 — Inyectar ConfigMap como archivo (patrón enterprise)

### 5.1 Crear ConfigMap tipo archivo

```
kubectl -n practice create configmap nginx-conf \
  --from-literal=custom.conf="server_name example.local;"
```

### 5.2 Montarlo como volumen

En el Deployment:

```
volumeMounts:
  - name: nginx-config
    mountPath: /etc/nginx/conf.d
volumes:
  - name: nginx-config
    configMap:
      name: nginx-conf
```

Apply:

```
kubectl apply -k .
```

---

## Paso 6 — Actualizar configuración SIN rebuild

### 6.1 Cambiar valor del ConfigMap

```
kubectl -n practice edit configmap web-config
```

Ejemplo:
```
APP_ENV=prod
```

### 6.2 Reiniciar pods (estrategia simple)

```
kubectl -n practice rollout restart deployment web-app
```

**Nota**
- ConfigMaps no reinician pods automáticamente
- En enterprise se usan:
  - checksum annotations
  - herramientas GitOps

---

## Paso 7 — Declarativo (forma correcta para enterprise)

### 7.1 Exportar YAML base

```
kubectl -n practice get configmap web-config -o yaml > web-config.yaml
kubectl -n practice get secret web-secret -o yaml > web-secret.yaml
```

Luego:
- borrar `metadata.creationTimestamp`
- borrar `resourceVersion`
- versionar en Git

---

## Errores comunes (y cómo explicarlos)

### E1) “Mi app no ve las variables”
Checklist:
```
kubectl -n practice describe pod <pod>
kubectl -n practice describe deploy web-app
```

Causa típica:
- typo en key del ConfigMap / Secret

---

### E2) “Edité ConfigMap pero no cambió nada”
Explicación:
- ConfigMap ≠ hot reload
- El pod debe reiniciarse

---

### E3) “Secret en GitHub”
Frase correcta:
> “Nunca versiono secretos en texto plano; uso vaults o sealed-secrets.”

---

## Cleanup del workshop

```
kubectl -n practice delete configmap web-config nginx-conf
kubectl -n practice delete secret web-secret
```

(O borra todo el namespace)

```
kubectl delete namespace practice --ignore-not-found=true
```

---

## Guion de entrevista (listo para usar)

- “Separé imagen de configuración.”
- “Uso ConfigMaps para config y Secrets para credenciales.”
- “Inyecto por env o volumen según el caso.”
- “Los cambios de config no requieren rebuild.”
- “Aplico GitOps para versionar configuración.”

---
