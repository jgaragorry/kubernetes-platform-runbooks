## Workshop 31 — ConfigMaps (configuración desacoplada)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **por qué no se hardcodea configuración en imágenes**
- Usar **ConfigMaps** como archivos y como variables de entorno
- Actualizar configuración **sin rebuild**
- Entender impacto en **rolling updates**
- Responder preguntas reales de **AKS / EKS / SRE**

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

## Concepto clave (entrevista)

> **ConfigMap = configuración NO sensible**  
> **Secret = información sensible**

Nunca mezclar ambos.

---

## Diagrama mental

```
ConfigMap
 ├── key=value (env)
 └── archivos (mount)
```

---

## Parte A — App SIN ConfigMap (anti-patrón)

Crear deployment básico:

```
kubectl -n practice create deployment web-hardcoded --image=nginx:1.27
```

El contenido de nginx está **dentro de la imagen**

❌ Cambiar config = rebuild + redeploy

---

## Parte B — Crear ConfigMap (key=value)

Crear ConfigMap manual:

```
kubectl -n practice create configmap web-config \
  --from-literal=APP_ENV=dev \
  --from-literal=APP_OWNER=platform-team
```

Ver:

```
kubectl -n practice get configmap web-config -o yaml
```

---

## Parte C — Usar ConfigMap como variables de entorno

Crear archivo:

```
vi deploy-env.yaml
```

Contenido:

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
            name: web-config
```

Aplicar:

```
kubectl apply -f deploy-env.yaml
```

Entrar al Pod:

```
kubectl -n practice exec -it <POD_NAME> -- env | grep APP_
```

Resultado esperado:

```
APP_ENV=dev
APP_OWNER=platform-team
```

---

## Parte D — ConfigMap como archivo (mount)

Crear archivo de config:

```
vi app.conf
```

Contenido:

```
APP_NAME=web-app
APP_VERSION=1.0.0
```

Crear ConfigMap desde archivo:

```
kubectl -n practice create configmap app-config \
  --from-file=app.conf
```

Ver:

```
kubectl -n practice get configmap app-config -o yaml
```

---

## Parte E — Montar ConfigMap como volumen

Crear archivo:

```
vi deploy-volume.yaml
```

Contenido:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-volume
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-volume
  template:
    metadata:
      labels:
        app: web-volume
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
          name: app-config
```

Aplicar:

```
kubectl apply -f deploy-volume.yaml
```

Entrar al Pod:

```
kubectl -n practice exec -it <POD_NAME> -- ls /etc/app
```

Ver contenido:

```
cat /etc/app/app.conf
```

---

## Parte F — Actualizar ConfigMap (cambio real)

Editar ConfigMap:

```
kubectl -n practice edit configmap app-config
```

Cambiar valor:

```
APP_VERSION=1.0.1
```

⚠️ **Importante**  
Los Pods **NO se actualizan automáticamente** en mounts.

---

## Parte G — Forzar reload de Pods

Opción común:

```
kubectl -n practice rollout restart deployment web-volume
```

Resultado:

- Nuevos Pods
- Config actualizada

---

## Parte H — Diferencias clave (tabla)

```
envFrom     → requiere restart
volumeMount → requiere restart
Secret      → datos sensibles
```

---

## Parte I — Buenas prácticas enterprise

✔ ConfigMaps versionados en Git  
✔ Uno por dominio lógico  
✔ Nombres claros  
✔ No meter passwords  

❌ No hardcodear  
❌ No usar ConfigMap para secretos  

---

## Parte J — Errores comunes

❌ Esperar hot-reload automático  
❌ Mezclar secretos  
❌ Cambiar config sin restart  

Debug checklist:

```
kubectl describe pod
kubectl get configmap
kubectl exec -- cat archivo
```

---

## Parte K — Preguntas típicas de entrevista

- ¿ConfigMap vs Secret?
- ¿Qué pasa si actualizo un ConfigMap?
- ¿Cómo forzar reload?
- ¿Por qué separar config del container?

---

## Parte L — Cleanup

```
kubectl -n practice delete deployment web-hardcoded web-env web-volume
kubectl -n practice delete configmap web-config app-config
```

---
