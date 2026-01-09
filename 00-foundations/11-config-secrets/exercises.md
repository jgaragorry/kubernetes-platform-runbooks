## Workshop 12 — ConfigMaps & Secrets (Configuración desacoplada)

---

## Objetivo (lo que vas a dominar)
Al finalizar este workshop podrás **diseñar, aplicar y depurar**:

- Diferencia **configuración vs código**
- Qué es un **ConfigMap** y cuándo usarlo
- Qué es un **Secret** y cómo se maneja (base64)
- Inyección de configuración como:
  - Variables de entorno
  - Archivos montados
- Buenas prácticas **enterprise / entrevistas**
- Errores comunes y debugging real

---

## Regla 0 — Contexto y estado base (NO saltar)

Verifica contexto:

```
kubectl config current-context
kubectl get nodes
```

Contexto esperado:

```
kind-k8s-lab
```

Namespace esperado:

```
practice
```

Si no existe:

```
kubectl create namespace practice
```

---

## Estado inicial requerido

Deployment base `web-app` (nginx) **ya existente**.

Verifica:

```
kubectl -n practice get deploy,pods
```

Debes ver:
- Deployment `web-app`
- 2 pods `Running`

---

## Concepto clave (mentalidad Kubernetes)

> **La imagen NO debe cambiar cuando cambia la configuración**

- Código → imagen
- Configuración → ConfigMap / Secret

Esto permite:
- Reutilización
- Rollbacks
- Promociones Dev → QA → Prod

---

## Parte A — ConfigMap (configuración NO sensible)

---

## A.1 Crear ConfigMap desde literal

Ejemplo: variables de aplicación

```
kubectl -n practice create configmap web-config \
  --from-literal=APP_ENV=dev \
  --from-literal=APP_OWNER=platform-team
```

Verificar:

```
kubectl -n practice get configmap
kubectl -n practice describe configmap web-config
```

---

## A.2 Usar ConfigMap como variables de entorno

### Editar Deployment

```
kubectl -n practice edit deployment web-app
```

Agrega dentro del contenedor:

```
env:
  - name: APP_ENV
    valueFrom:
      configMapKeyRef:
        name: web-config
        key: APP_ENV
  - name: APP_OWNER
    valueFrom:
      configMapKeyRef:
        name: web-config
        key: APP_OWNER
```

Guarda y cierra.

---

## A.3 Verificar inyección

Espera rollout:

```
kubectl -n practice rollout status deployment web-app
```

Ejecuta dentro de un pod:

```
kubectl -n practice exec -it deploy/web-app -- env | grep APP_
```

Resultado esperado:

```
APP_ENV=dev
APP_OWNER=platform-team
```

---

## Parte B — ConfigMap como archivo (patrón enterprise)

---

## B.1 Crear ConfigMap desde archivo

Crear archivo local:

```
cat <<EOF > app.conf
server_name=web-app
log_level=info
EOF
```

Crear ConfigMap:

```
kubectl -n practice create configmap web-config-file \
  --from-file=app.conf
```

Verificar:

```
kubectl -n practice describe configmap web-config-file
```

---

## B.2 Montar ConfigMap como volumen

Editar deployment:

```
kubectl -n practice edit deployment web-app
```

Agregar:

```
volumeMounts:
  - name: config-volume
    mountPath: /etc/app

volumes:
  - name: config-volume
    configMap:
      name: web-config-file
```

---

## B.3 Verificar archivo dentro del pod

```
kubectl -n practice exec -it deploy/web-app -- ls /etc/app
kubectl -n practice exec -it deploy/web-app -- cat /etc/app/app.conf
```

---

## Parte C — Secrets (configuración sensible)

---

## Concepto clave

> **Secrets NO cifran por defecto**
>  
> Kubernetes los guarda en base64  
> El cifrado real depende del cluster

---

## C.1 Crear Secret desde literal

```
kubectl -n practice create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=SuperSecret123
```

Verificar:

```
kubectl -n practice get secret
kubectl -n practice describe secret db-secret
```

---

## C.2 Ver contenido base64 (educativo)

```
kubectl -n practice get secret db-secret -o yaml
```

Decodificar manualmente:

```
echo U3VwZXJTZWNyZXQxMjM= | base64 -d
```

---

## C.3 Usar Secret como variable de entorno

Editar deployment:

```
kubectl -n practice edit deployment web-app
```

Agregar:

```
env:
  - name: DB_USER
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_USER
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: DB_PASSWORD
```

---

## C.4 Verificar Secret en el pod

```
kubectl -n practice exec -it deploy/web-app -- env | grep DB_
```

---

## Parte D — Buenas prácticas enterprise (ENTREVISTA)

---

## Qué SÍ hacer

- ConfigMap para:
  - URLs
  - Flags
  - Nombres
- Secret para:
  - Passwords
  - Tokens
  - API Keys
- Usar:
  - External Secrets (Vault, AWS SM, Azure KV)
- Versionar manifests
- Separar por entorno

---

## Qué NO hacer

- ❌ Secrets en Git
- ❌ Hardcodear credenciales
- ❌ Reusar Secrets entre namespaces
- ❌ Usar ConfigMap para passwords

---

## Debugging rápido (runbook)

```
kubectl -n practice get configmap
kubectl -n practice get secret
kubectl -n practice describe pod <pod>
kubectl -n practice logs <pod>
```

---

## Cleanup (opcional)

```
kubectl -n practice delete configmap web-config web-config-file
kubectl -n practice delete secret db-secret
```

---

