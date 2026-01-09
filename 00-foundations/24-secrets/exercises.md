## Workshop 24 — Secrets (Gestión segura de credenciales)

---

## Objetivo (nivel enterprise / entrevistas)

Al finalizar este workshop podrás:

- Explicar **qué es un Secret y para qué se usa**
- Diferenciar **ConfigMap vs Secret**
- Crear Secrets:
  - desde línea de comandos
  - desde archivos
- Inyectar Secrets:
  - como **variables de entorno**
  - como **archivos**
- Entender **limitaciones de seguridad reales**
- Responder preguntas típicas de entrevistas (AKS / EKS / GKE)

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

> **Secret ≠ cifrado fuerte por defecto**

Por defecto:
- Kubernetes **codifica en base64**
- NO cifra automáticamente en etcd (a menos que lo configures)

👉 Aun así:
- evita hardcodear secretos en imágenes
- habilita rotación
- desacopla credenciales del código

---

## Parte A — Comparación rápida (entrevista)

| Recurso    | Uso                        | Seguridad |
|-----------|----------------------------|-----------|
| ConfigMap | Configuración no sensible  | Texto plano |
| Secret    | Datos sensibles            | Base64 |

---

## Parte B — Crear Secret desde literales

Crear Secret genérico:

```
kubectl -n practice create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=SuperSecret123 \
  --from-literal=DB_NAME=appdb
```

Verificar:

```
kubectl -n practice get secrets
kubectl -n practice describe secret db-secret
```

⚠️ Nota:
> Kubernetes **oculta el valor**, pero sigue existiendo.

---

## Parte C — Ver contenido real (base64)

Obtener valor codificado:

```
kubectl -n practice get secret db-secret -o yaml
```

Decodificar manualmente:

```
echo "U3VwZXJTZWNyZXQxMjM=" | base64 --decode
```

---

## Parte D — Usar Secret como variables de entorno

Crear deployment:

```
kubectl -n practice create deployment web-secret \
  --image=nginx:1.27
```

Editar deployment:

```
kubectl -n practice edit deployment web-secret
```

Agregar en el container:

```
envFrom:
  - secretRef:
      name: db-secret
```

Verificar Pod:

```
kubectl -n practice exec -it <pod-name> -- sh
```

Dentro del Pod:

```
env | grep DB_
```

Salir:

```
exit
```

---

## Parte E — Secret desde archivo (caso real)

Crear archivo local:

```
cat <<EOF > credentials.txt
username=admin
password=SuperSecret123
EOF
```

Crear Secret:

```
kubectl -n practice create secret generic file-secret \
  --from-file=credentials.txt
```

Verificar:

```
kubectl -n practice describe secret file-secret
```

---

## Parte F — Montar Secret como archivo

Editar deployment:

```
kubectl -n practice edit deployment web-secret
```

Agregar volumen:

```
volumes:
  - name: secret-volume
    secret:
      secretName: file-secret
```

Y en el container:

```
volumeMounts:
  - name: secret-volume
    mountPath: /etc/secrets
    readOnly: true
```

Verificar:

```
kubectl -n practice exec -it <pod-name> -- sh
```

Dentro:

```
ls /etc/secrets
cat /etc/secrets/credentials.txt
```

---

## Parte G — Buenas prácticas enterprise

✔ Usar Secrets para:
- passwords
- tokens
- API keys
- certificados

✔ Recomendado:
- RBAC restrictivo
- rotación periódica
- External Secrets (Vault / AWS Secrets Manager / Azure Key Vault)

❌ NO:
- commitear Secrets en Git
- exponerlos por logs
- usarlos como ConfigMaps

---

## Parte H — Errores comunes

❌ Error:
> “No veo mis variables”

Checklist:

```
kubectl get secret -n practice
kubectl describe pod <pod-name>
kubectl rollout status deployment web-secret
```

Causas típicas:
- typo en nombre del Secret
- namespace incorrecto
- Pod no reiniciado
- key inexistente

---

## Parte I — Preguntas de entrevista

**Pregunta:**
> ¿Kubernetes cifra Secrets por defecto?

**Respuesta senior:**
> “No. Por defecto están en base64; el cifrado en etcd debe configurarse explícitamente.”

---

## Parte J — Cleanup (opcional)

```
kubectl -n practice delete deployment web-secret
kubectl -n practice delete secret db-secret
kubectl -n practice delete secret file-secret
```

---
