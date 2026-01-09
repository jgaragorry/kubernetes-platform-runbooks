## Workshop 9 — ConfigMaps y Secrets (configuración externa y 12-Factor)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **diseñar, implementar y explicar**:

- Diferencia **real** entre:
  - `ConfigMap`
  - `Secret`
- Por qué **NO** van hardcodeados en el Deployment
- Cómo inyectar configuración vía:
  - variables de entorno
  - archivos montados
- Buenas prácticas **enterprise** (12-Factor App)
- Cómo responder **preguntas de entrevista** sobre seguridad y configuración

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
- **Sin ConfigMaps ni Secrets**

Verifica:

```
kubectl -n practice get configmaps,secrets
```

Resultado esperado:
- Solo secrets del sistema (default-token)

---

## Conceptos clave (ANTES de tocar YAML)

### Qué es un ConfigMap
Un **ConfigMap** almacena:
- configuración **NO sensible**
- texto plano
- parámetros de runtime

Ejemplos:
- URLs
- flags
- feature toggles
- mensajes

---

### Qué es un Secret
Un **Secret** almacena:
- datos **sensibles**
- base64 (no cifrado por defecto)
- credenciales

Ejemplos:
- passwords
- tokens
- API keys
- certificados

---

## Regla de oro (12-Factor)
> “El código no cambia entre entornos, la configuración sí.”

---

## Paso 1 — Crear ConfigMap (forma declarativa)

### 1.1 Crear archivo

```
mkdir -p 00-foundations/08-configmaps-secrets
cd 00-foundations/08-configmaps-secrets
```

Archivo `configmap-app.yaml`:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: practice
data:
  APP_ENV: "dev"
  APP_MESSAGE: "Hola desde ConfigMap"
```

---

### 1.2 Aplicar ConfigMap

```
kubectl apply -f configmap-app.yaml
```

Verificar:

```
kubectl -n practice get configmap web-config -o yaml
```

---

## Paso 2 — Inyectar ConfigMap como variables de entorno

### 2.1 Editar Deployment

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
- name: APP_MESSAGE
  valueFrom:
    configMapKeyRef:
      name: web-config
      key: APP_MESSAGE
```

Guardar y salir.

---

### 2.2 Validar variables dentro del pod

```
kubectl -n practice exec deploy/web-app -- printenv | grep APP_
```

Resultado esperado:

```
APP_ENV=dev
APP_MESSAGE=Hola desde ConfigMap
```

---

## Paso 3 — Montar ConfigMap como archivo

### 3.1 Editar Deployment (volumen)

Agrega:

```
volumes:
- name: config-volume
  configMap:
    name: web-config
```

En el contenedor:

```
volumeMounts:
- name: config-volume
  mountPath: /etc/config
```

---

### 3.2 Ver archivos

```
kubectl -n practice exec deploy/web-app -- ls /etc/config
kubectl -n practice exec deploy/web-app -- cat /etc/config/APP_MESSAGE
```

---

## Paso 4 — Crear Secret (forma segura)

### 4.1 Crear Secret vía YAML

Archivo `secret-app.yaml`:

```
apiVersion: v1
kind: Secret
metadata:
  name: web-secret
  namespace: practice
type: Opaque
data:
  DB_PASSWORD: c3VwZXJzZWNyZXQ=
```

(Valor real: `supersecret` en base64)

Aplicar:

```
kubectl apply -f secret-app.yaml
```

---

### 4.2 Verificar Secret

```
kubectl -n practice get secret web-secret
```

---

## Paso 5 — Inyectar Secret como variable de entorno

Editar Deployment:

```
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: web-secret
      key: DB_PASSWORD
```

---

### 5.1 Validar Secret

```
kubectl -n practice exec deploy/web-app -- printenv | grep DB_PASSWORD
```

---

## Paso 6 — Montar Secret como archivo

Agrega volumen:

```
volumes:
- name: secret-volume
  secret:
    secretName: web-secret
```

En el contenedor:

```
volumeMounts:
- name: secret-volume
  mountPath: /etc/secret
  readOnly: true
```

Verificar:

```
kubectl -n practice exec deploy/web-app -- ls /etc/secret
```

---

## Tabla resumen (memorizable)

| Recurso | Sensible | Editable | Uso |
|------|---------|---------|-----|
| ConfigMap | ❌ | Fácil | Configuración |
| Secret | ✅ | Controlado | Credenciales |

---

## Buenas prácticas enterprise

- ❌ No hardcodear valores en YAML
- ❌ No versionar Secrets reales
- ✅ Usar GitOps + External Secrets
- ✅ Rotar Secrets
- ✅ Separar config por entorno

---

## Frases de entrevista (golden answers)

- “ConfigMap es config, Secret es credencial.”
- “Nunca versiono secretos reales.”
- “La app no debe saber en qué entorno corre.”
- “Config externa permite despliegues idénticos.”

---

## Errores comunes

### ❌ Guardar passwords en ConfigMap
### ❌ Subir secrets reales a GitHub
### ❌ Reiniciar app para cada cambio menor
### ❌ No usar envFrom cuando aplica

---

## Cleanup (opcional)

```
kubectl delete namespace practice --ignore-not-found=true
```

---
