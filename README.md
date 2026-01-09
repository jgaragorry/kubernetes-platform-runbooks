# Kubernetes Platform Runbooks (Enterprise)

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-000000?style=for-the-badge&logo=linux&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![DevOps](https://img.shields.io/badge/DevOps-0A0A0A?style=for-the-badge&logo=azuredevops&logoColor=white)
![SRE](https://img.shields.io/badge/SRE-4CAF50?style=for-the-badge)
![Enterprise](https://img.shields.io/badge/Enterprise-222222?style=for-the-badge)

---

## 📌 Propósito del Repositorio

Este repositorio es una **plataforma de conocimiento profesional y enterprise-ready** diseñada para:

- Documentar **runbooks reales de Kubernetes**
- Construir **dominio técnico progresivo**, desde fundamentos hasta escenarios avanzados
- Servir como **base de entrevistas técnicas (AKS / EKS / Kubernetes puro)**
- Actuar como **librería personal de referencia**, troubleshooting y buenas prácticas
- Demostrar **madurez operativa**, no solo uso básico de Kubernetes

No es un tutorial básico.  
Es un **sistema estructurado de aprendizaje y operación**, similar a lo que usan equipos SRE / Platform / Cloud en producción.

---

## 🎯 Objetivos Técnicos

Al completar este repositorio serás capaz de:

- Operar Kubernetes **con y sin mirar documentación**
- Entender **qué recurso usar, cuándo y por qué**
- Explicar decisiones técnicas en entrevistas senior
- Detectar errores comunes antes de que ocurran
- Traducir Kubernetes local (kind) a **AKS / EKS / GKE**
- Diseñar workloads resilientes, observables y seguros

---

## 🧠 Enfoque Metodológico

Este repo sigue un enfoque **práctico, progresivo y repetible**:

1. Aprender haciendo
2. Repetir hasta automatizar
3. Documentar cada paso
4. Entender el porqué, no solo el cómo
5. Pensar como plataforma, no como usuario final

Cada comando ejecutado aquí tiene:
- un objetivo
- un impacto
- un motivo de ejecución
- una equivalencia en entornos enterprise

---

## 🏗️ Arquitectura de Aprendizaje

- Kubernetes local con kind
- Docker como runtime
- Linux como base operativa
- kubectl como herramienta principal
- Manifests YAML progresivos
- Sin magia, sin abstraer conceptos clave

---

## 📂 Estructura del Repositorio

    kubernetes-platform-runbooks/
    ├── README.md
    ├── LICENSE
    ├── docs/
    │   ├── conventions.md
    │   ├── kubectl-command-matrix.md
    │   ├── interview-notes.md
    │   └── troubleshooting.md
    │
    ├── 00-foundations/
    │   ├── 00-environment/
    │   │   ├── bootstrap-k8s-lab.sh
    │   │   └── README.md
    │   ├── 01-kind-cluster/
    │   │   ├── create-cluster.sh
    │   │   └── README.md
    │   └── 02-kubectl-basics/
    │       ├── exercises.md
    │       └── README.md
    │
    ├── 10-workloads/
    │   ├── pods/
    │   ├── deployments/
    │   ├── replicasets/
    │   └── services/
    │
    ├── 20-networking/
    │   ├── clusterip/
    │   ├── nodeport/
    │   └── ingress/
    │
    ├── 30-storage/
    │   ├── volumes/
    │   └── pvc/
    │
    ├── 40-configuration/
    │   ├── configmaps/
    │   └── secrets/
    │
    ├── 50-observability/
    │   ├── probes/
    │   ├── metrics/
    │   └── logs/
    │
    ├── 60-security/
    │   ├── rbac/
    │   ├── serviceaccounts/
    │   └── networkpolicies/
    │
    ├── 70-scaling/
    │   ├── hpa/
    │   └── autoscaling/
    │
    └── 99-cleanup/
        └── destroy-lab.sh

---

## 🧪 Entorno Base

Este repositorio asume:

- Ubuntu Server 24.04 LTS
- Docker instalado y operativo
- kubectl instalado
- kind como Kubernetes local
- Usuario Linux no root con sudo

---

## 🧩 Kubernetes Distribution Strategy

| Entorno | Objetivo |
|--------|----------|
| kind   | Aprendizaje profundo y control total |
| AKS    | Traducción directa a Azure |
| EKS    | Traducción directa a AWS |
| GKE    | Referencia conceptual |

Lo aprendido aquí **aplica directamente** a todos ellos.

---

## 🎓 Valor para Entrevistas

Este repo permite responder con seguridad preguntas como:

- ¿Diferencia entre Pod y Deployment?
- ¿Qué pasa si se cae un Pod?
- ¿Cómo garantizas alta disponibilidad?
- ¿Cómo expones una app internamente?
- ¿Cómo migras esto a AKS?
- ¿Cómo debuggeas un servicio que no responde?

Con **experiencia real**, no teoría.

---

## 🚦 Estado del Repositorio

- En construcción progresiva
- Enfocado en excelencia técnica
- Diseñado como base de conocimiento viva
- Nivel profesional / enterprise

---

## 📬 Autor

- GitHub: https://github.com/jgaragorry
- LinkedIn: https://www.linkedin.com/in/jgaragorry/

---

## ✅ Regla Principal

Si no puedes explicarlo sin mirar documentación, aún no lo dominas.  
Este repositorio existe para cambiar eso.

