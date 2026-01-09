## Workshop 17 — Secrets (Gestión segura de credenciales en Kubernetes)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué es un Secret** y **por qué existe**
- Diferenciar **ConfigMap vs Secret** con criterio profesional
- Crear Secrets de **3 formas**
- Consumir Secrets:
  - como variables de entorno
  - como archivos montados
- Entender **riesgos reales** (base64 ≠ cifrado)
- Explicar **cómo se maneja en producción**
- Identificar **errores comunes en entrevistas**
- Prepararte para AKS / EKS / GKE

---

## Regla 0 — Contexto correcto

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

> **Secret ≠ cifrado**
>
> Kubernetes Secrets están **codificados en base64**,  
> **NO cifrados por defecto**.
>
> El cifrado real depende del cluster:
> - etcd encryption
> - KMS (AKS / EKS / GKE)
> - herramientas externas (Vault, SOPS)

---

## Parte A — Qué es un Secret

Un Secret es:

- Un objeto Kubernetes
- Guarda **datos sensibles**
- Usado para:
  - passwords
  - tokens
  - API keys
  - certificados
- Se expone **solo al Pod que lo necesita**

Tipos comunes:

- `Opaque` (genérico)
- `kubernetes.io/dockerconfigjson`
- `tls`

---

## Parte B — Crear Secret desde CLI (key/value)

### B.1 Crear Secret Opaque

```
kubectl -n practice create secret generic app-secret \
  --from-literal=DB_USER=appuser \
  --from-literal=DB_PASSWORD=SuperSecret123 \
  --from-literal=API_TOKEN=abc123xyz
```

Verifica:

```
kubectl -n practice get secrets
kubectl -n practice describe secret app-secret
```

⚠️ Observación:

- Kubernetes **NO muestra valores**
- Solo metadata

---

### B.2 Ver contenido real (educativo)

```
kubectl -n practice get secret app-secret -o yaml
```

Verás base64:

```
DB_PASSWORD: U3VwZXJTZWNyZXQxMjM=
```

Decodificar (solo para aprender):

```
echo U3VwZXJTZWNyZXQxMjM= | base64 -d
```

---

## Parte C — Usar Secret como variables de entorno

### C.1 Deployment usando Secret

Archivo: `deployment-secret-env.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-secret-env
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-secret-env
  template:
    metadata:
      labels:
        app: web-secret-env
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        envFrom:
        - secretRef:
            name: app-secret
```

Aplicar:

```
kubectl apply -f deployment-secret-env.yaml
```

---

### C.2 Ver variables en el Pod

```
kubectl -n practice exec -it pod/<POD_NAME> -- env | grep DB_
```

Salida esperada:

```
DB_USER=appuser
DB_PASSWORD=SuperSecret123
```

---

## Parte D — Secret como archivo (patrón enterprise)

### D.1 Crear Secret desde archivo

Crea archivo `credentials.conf`:

```
username=service_user
password=S3rv1c3P@ss
```

Crear Secret:

```
kubectl -n practice create secret generic app-file-secret \
  --from-file=credentials.conf
```

---

### D.2 Montar Secret como volumen

Archivo: `deployment-secret-file.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-secret-file
  namespace: practice
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-secret-file
  template:
    metadata:
      labels:
        app: web-secret-file
    spec:
      containers:
      - name: nginx
        image: nginx:1.27
        volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: secret-volume
        secret:
          secretName: app-file-secret
```

Aplicar:

```
kubectl apply -f deployment-secret-file.yaml
```

---

### D.3 Ver archivo dentro del Pod

```
kubectl -n practice exec -it pod/<POD_NAME> -- ls /etc/secrets
kubectl -n practice exec -it pod/<POD_NAME> -- cat /etc/secrets/credentials.conf
```

---

## Parte E — Actualizar Secrets (realidad operativa)

- Cambiar Secret **NO reinicia Pods**
- Variables de entorno:
  - requieren restart
- Volúmenes:
  - reflejan cambios automáticamente (con delay)

Restart recomendado:

```
kubectl -n practice rollout restart deployment web-secret-env
```

---

## Parte F — Errores comunes (pregunta típica)

❌ Errores:

- Guardar secrets en Git
- Usar ConfigMap para passwords
- Pensar que base64 es cifrado
- Exponer secrets en logs

✅ Buenas prácticas:

- RBAC estricto
- Namespaces aislados
- External Secrets (Vault, Azure Key Vault, AWS Secrets Manager)
- GitOps con SOPS

---

## Parte G — ConfigMap vs Secret (respuesta de senior)

| Aspecto | ConfigMap | Secret |
|---|---|---|
| Sensible | No | Sí |
| Base64 | No | Sí |
| Git | Permitido | No |
| Seguridad | Básica | Reforzada |
| Uso | Config | Credenciales |

---

## Cómo explicarlo en entrevista (respuesta modelo)

> “Usamos Secrets para manejar credenciales de forma segura,
> evitando hardcodeo, con RBAC y cifrado en etcd o KMS,
> alineado a buenas prácticas enterprise y zero trust.”

---

## Cleanup (opcional)

```
kubectl -n practice delete deployment web-secret-env web-secret-file
kubectl -n practice delete secret app-secret app-file-secret
```

---

