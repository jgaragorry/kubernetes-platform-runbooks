## Workshop 16 — ConfigMaps & Environment Variables (Configuración desacoplada)

---

## Objetivo (dominio real que vas a ganar)

Al finalizar este workshop podrás:

- Explicar **qué es un ConfigMap** y **qué NO es**
- Entender **por qué la configuración NO va en la imagen**
- Usar ConfigMaps de **3 formas distintas**
- Inyectar configuración:
  - como variables de entorno
  - como archivos
- Explicar **12-Factor App** en entrevistas
- Diferenciar **ConfigMap vs Secret**
- Aplicar cambios **sin reconstruir imágenes**
- Entender qué se usa **en producción enterprise**

---

## Regla 0 — Contexto correcto (obligatorio)

Verifica que estás en el cluster correcto:

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

## Concepto clave (muy importante)

> **Las imágenes NO deben contener configuración**
>
> La configuración cambia más que el código.
>
> Kubernetes separa:
> - Código (imagen)
> - Configuración (ConfigMap / Secret)

---

## Parte A — Qué es un ConfigMap

Un ConfigMap es:

- Un objeto Kubernetes
- Guarda **configuración NO sensible**
- Puede contener:
  - pares key/value
  - archivos completos (texto)

Ejemplos reales:
- URLs
- flags
- nombres de entorno
- configuraciones de apps
- archivos `.conf`

---

## Parte B — Crear ConfigMap simple (key/value)

### B.1 Crear ConfigMap desde CLI

```
kubectl -n practice create configmap app-config \
  --from-literal=APP_NAME=web-app \
  --from-literal=APP_ENV=dev \
  --from-literal=APP_OWNER=platform-team
```

Verifica:

```
kubectl -n practice get configmap
kubectl -n practice describe configmap app-config
```

---

## Parte C — Usar ConfigMap como variables de entorno

---

### C.1 Crear Deployment que consuma ConfigMap

Archivo: `deployment-env.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-env
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-env
  template:
    metadata:
      labels:
        app: web-env
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        envFrom:
        - configMapRef:
            name: app-config
```

Aplicar:

```
kubectl apply -f deployment-env.yaml
```

Verifica Pod:

```
kubectl -n practice get pods
```

---

### C.2 Ver variables dentro del Pod

```
kubectl -n practice exec -it pod/<POD_NAME> -- env | grep APP_
```

Salida esperada:

```
APP_NAME=web-app
APP_ENV=dev
APP_OWNER=platform-team
```

---

## Parte D — ConfigMap como archivo (caso real)

### D.1 Crear ConfigMap desde archivo

Crea archivo `app.conf`:

```
APP_MODE=production
LOG_LEVEL=info
FEATURE_X=true
```

Crear ConfigMap:

```
kubectl -n practice create configmap app-file-config \
  --from-file=app.conf
```

Verifica:

```
kubectl -n practice describe configmap app-file-config
```

---

### D.2 Montar ConfigMap como volumen

Archivo: `deployment-file.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-file
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-file
  template:
    metadata:
      labels:
        app: web-file
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
      volumes:
      - name: config-volume
        configMap:
          name: app-file-config
```

Aplicar:

```
kubectl apply -f deployment-file.yaml
```

---

### D.3 Ver archivo dentro del Pod

```
kubectl -n practice exec -it pod/<POD_NAME> -- ls /etc/app-config
kubectl -n practice exec -it pod/<POD_NAME> -- cat /etc/app-config/app.conf
```

---

## Parte E — Actualizar ConfigMap (comportamiento real)

Editar ConfigMap:

```
kubectl -n practice edit configmap app-config
```

⚠️ Importante:

- Variables de entorno **NO se actualizan en Pods existentes**
- Volúmenes **sí reflejan cambios**

👉 En producción:
- Se reinician Pods (rolling restart)

Ejemplo:

```
kubectl -n practice rollout restart deployment web-env
```

---

## Parte F — ConfigMap vs Secret (tabla entrevista)

| Aspecto | ConfigMap | Secret |
|---|---|---|
| Sensibilidad | NO | SÍ |
| Base64 | NO | SÍ |
| Uso | Config | Passwords, tokens |
| Git | Puede | Nunca |

---

## Parte G — Buenas prácticas enterprise

- Un ConfigMap por aplicación
- Versionar YAML (GitOps)
- No hardcodear valores
- Separar config por entorno
- Nunca meter secretos en ConfigMap

---

## Cómo explicarlo en entrevista (respuesta modelo)

> “Usamos ConfigMaps para desacoplar configuración del código,
> permitiendo cambios sin rebuild de imágenes, alineados al
> modelo 12-Factor y prácticas GitOps.”

---

## Cleanup (opcional)

```
kubectl -n practice delete deployment web-env web-file
kubectl -n practice delete configmap app-config app-file-config
```

---

