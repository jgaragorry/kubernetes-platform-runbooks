## Workshop 23 — ConfigMaps (Configuración desacoplada)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué es un ConfigMap y para qué se usa**
- Separar **configuración vs código**
- Inyectar configuración en Pods:
  - como **variables de entorno**
  - como **archivos**
- Cambiar configuración **sin reconstruir imágenes**
- Responder preguntas típicas en entrevistas (AKS / EKS / GKE)

---

## Regla 0 — Contexto y namespace

Verifica contexto activo:

```
kubectl config current-context
```

Resultado esperado:

```
kind-k8s-lab
```

Namespace de trabajo:

```
practice
```

---

## Concepto clave (muy importante)

> **La imagen del contenedor NO debe cambiar**
>
> La configuración:
> - cambia entre entornos
> - vive fuera del código
> - se versiona aparte

👉 **ConfigMap = configuración no sensible**

---

## Parte A — Problema sin ConfigMaps

Escenario clásico (mal diseño):

- Imagen contiene:
  - puerto
  - hostname
  - environment
- Para cambiar algo → **rebuild + redeploy**

❌ Anti-pattern enterprise

---

## Parte B — Crear ConfigMap simple (key/value)

Crear ConfigMap:

```
kubectl -n practice create configmap app-config \
  --from-literal=APP_ENV=dev \
  --from-literal=APP_NAME=web-app \
  --from-literal=APP_VERSION=1.0
```

Verificar:

```
kubectl -n practice get configmap
kubectl -n practice describe configmap app-config
```

---

## Parte C — Usar ConfigMap como variables de entorno

Deployment usando envFrom:

```
kubectl -n practice create deployment web-config \
  --image=nginx:1.27
```

Editar deployment:

```
kubectl -n practice edit deployment web-config
```

Agregar dentro del container:

```
envFrom:
  - configMapRef:
      name: app-config
```

Guardar y salir.

Ver Pods:

```
kubectl -n practice get pods
```

Entrar a un Pod:

```
kubectl -n practice exec -it <pod-name> -- sh
```

Ver variables:

```
env | grep APP_
```

Salir:

```
exit
```

---

## Parte D — ConfigMap como archivo

Crear ConfigMap desde archivo:

```
cat <<EOF > app.conf
server_name=web-app
server_env=dev
server_version=1.0
EOF
```

Crear ConfigMap:

```
kubectl -n practice create configmap app-config-file \
  --from-file=app.conf
```

Verificar:

```
kubectl -n practice describe configmap app-config-file
```

---

## Parte E — Montar ConfigMap como volumen

Editar deployment:

```
kubectl -n practice edit deployment web-config
```

Agregar volumen:

```
volumes:
  - name: config-volume
    configMap:
      name: app-config-file
```

Y en el container:

```
volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```

Verificar:

```
kubectl -n practice exec -it <pod-name> -- sh
```

Dentro:

```
ls /etc/config
cat /etc/config/app.conf
```

---

## Parte F — Cambio de configuración (sin rebuild)

Actualizar ConfigMap:

```
kubectl -n practice edit configmap app-config
```

Cambiar:

```
APP_ENV=prod
```

⚠️ Nota importante:

- Variables env → requieren restart del Pod
- Archivos montados → se actualizan automáticamente

Forzar rollout:

```
kubectl -n practice rollout restart deployment web-config
```

---

## Parte G — Buenas prácticas enterprise

✔ Usar ConfigMap para:
- URLs
- flags
- puertos
- nombres de servicios

❌ NO usar para:
- passwords
- tokens
- claves privadas

👉 Para eso existe **Secret** (Workshop 24)

---

## Parte H — Error común (muy real)

❌ Error:
> “Mi ConfigMap no se refleja”

Checklist:

```
kubectl get configmap -n practice
kubectl describe pod <pod-name>
kubectl rollout status deployment web-config
```

Causas típicas:
- typo en nombre
- namespace incorrecto
- Pod no reiniciado
- key inexistente

---

## Parte I — Preguntas típicas de entrevista

**Pregunta:**
> ¿Diferencia entre ConfigMap y Secret?

**Respuesta senior:**
> “ConfigMap maneja configuración no sensible; Secret maneja datos sensibles y se codifica en base64.”

---

## Parte J — Cleanup (opcional)

```
kubectl -n practice delete deployment web-config
kubectl -n practice delete configmap app-config
kubectl -n practice delete configmap app-config-file
```

---
