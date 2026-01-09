## Workshop 28 — ConfigMaps & Secrets (Configuración Desacoplada)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué la configuración NO debe ir en la imagen**
- Diferenciar **ConfigMap vs Secret**
- Inyectar configuración como:
  - variables de entorno
  - archivos montados
- Responder preguntas reales de **AKS / EKS / producción**

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

Si no existe:

```
kubectl create namespace practice
```

---

## Concepto clave (muy importante)

> **La imagen es inmutable, la configuración es mutable**

En Kubernetes:
- **La app NO debe cambiar**
- **La configuración SÍ cambia**

Por eso existen:
- ConfigMaps → configuración no sensible
- Secrets → datos sensibles

---

## Diagrama mental (entrevista)

```
Deployment
 ├── Image (nginx, app)
 ├── ConfigMap → ENV / Files
 └── Secret → ENV / Files
```

---

## Parte A — Crear ConfigMap (texto plano)

### ConfigMap con variables simples

```
kubectl -n practice create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=APP_REGION=us-east-1
```

Verificar:

```
kubectl -n practice get configmap app-config -o yaml
```

---

## Parte B — Crear Secret (datos sensibles)

### Secret tipo Opaque

```
kubectl -n practice create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=supersecret
```

Verificar (nota: base64):

```
kubectl -n practice get secret app-secret -o yaml
```

---

## Parte C — Deployment usando ENV desde ConfigMap y Secret

Crear archivo:

```
vi deploy-env.yaml
```

Contenido:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: env-demo
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: env-demo
  template:
    metadata:
      labels:
        app: env-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: APP_REGION
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_REGION
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: DB_PASSWORD
```

Aplicar:

```
kubectl apply -f deploy-env.yaml
```

---

## Parte D — Ver variables dentro del Pod

Obtener Pod:

```
kubectl -n practice get pods
```

Entrar al Pod:

```
kubectl -n practice exec -it <POD_NAME> -- env | grep APP
```

Resultado esperado:

```
APP_ENV=production
APP_REGION=us-east-1
```

Ver secret (ojo entrevista):

```
kubectl -n practice exec -it <POD_NAME> -- env | grep DB_
```

✔ Kubernetes inyecta secrets como texto plano **dentro del contenedor**

---

## Parte E — ConfigMap como archivo (muy usado)

### Crear ConfigMap desde archivo

```
cat <<EOF > app.conf
server_name=myapp
log_level=info
EOF
```

```
kubectl -n practice create configmap file-config \
  --from-file=app.conf
```

---

### Deployment montando archivo

```
vi deploy-volume.yaml
```

Contenido:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-demo
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: volume-demo
  template:
    metadata:
      labels:
        app: volume-demo
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: config-volume
          mountPath: /etc/app
      volumes:
      - name: config-volume
        configMap:
          name: file-config
```

Aplicar:

```
kubectl apply -f deploy-volume.yaml
```

---

## Parte F — Ver archivo montado

```
kubectl -n practice exec -it <POD_NAME> -- ls /etc/app
```

```
kubectl -n practice exec -it <POD_NAME> -- cat /etc/app/app.conf
```

✔ ConfigMap convertido en archivo

---

## Parte G — Qué aprendiste (entrevista)

✔ ConfigMaps:
- config desacoplada
- no sensible
- versionable

✔ Secrets:
- datos sensibles
- base64 (no cifrado por defecto)
- integrables con KMS (cloud)

✔ Usos reales:
- strings de conexión
- feature flags
- parámetros runtime

---

## Parte H — Errores comunes

❌ Cambiar config dentro del contenedor  
❌ Hardcodear passwords  
❌ Committear secrets en Git  

Checklist debug:

```
kubectl describe pod <pod>
kubectl get configmap
kubectl get secret
```

---

## Parte I — Buenas prácticas enterprise

✔ Usar:
- External Secrets
- Azure Key Vault / AWS Secrets Manager
- CSI Secret Store

✔ Rotar secrets sin redeploy completo

✔ Configuración por environment:
- dev
- qa
- prod

---

## Parte J — Preguntas típicas de entrevista

- ¿ConfigMap reinicia Pods al cambiar?
- ¿Secret está cifrado?
- ¿Cómo rotar secrets?
- ¿Diferencia ENV vs volumen?

Debes responder con este lab.

---

## Parte K — Cleanup (opcional)

```
kubectl -n practice delete deployment env-demo volume-demo
kubectl -n practice delete configmap app-config file-config
kubectl -n practice delete secret app-secret
```

---

