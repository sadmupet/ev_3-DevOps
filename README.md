# Tienda de Perritos — CI/CD con GitHub Actions y despliegue en Amazon EKS

Proyecto para la Evaluación Final Transversal de **ISY1101 - Introducción a Herramientas DevOps**.

Aplicación de 3 capas (frontend + backend + base de datos relacional) para gestionar el catálogo de
productos de una tienda de alimentos para perritos, con CI/CD automatizado y despliegue en un clúster
de **Amazon EKS**.

## Arquitectura

```
                         ┌─────────────────────────┐
 Usuario ── HTTP ──────► │  Service tienda-frontend │  (LoadBalancer / ELB)
                         │   Nginx + HTML/JS         │
                         └────────────┬──────────────┘
                                      │ proxy /api/*
                                      ▼
                         ┌─────────────────────────┐
                         │  Service tienda-backend  │  (ClusterIP)
                         │   Node.js + Express      │
                         └────────────┬──────────────┘
                                      │ mysql2
                                      ▼
                         ┌─────────────────────────┐
                         │   Service tienda-db      │  (ClusterIP headless)
                         │   MySQL 8                │
                         └─────────────────────────┘

 Namespace de Kubernetes: "tienda"
 HPA en frontend y backend (autoescalado por % de CPU)
 Imágenes publicadas en Amazon ECR
```

## Stack

- **Frontend:** HTML + JS vanilla, servido con Nginx (proxy inverso hacia el backend)
- **Backend:** Node.js 18 + Express, API REST CRUD de productos
- **Base de datos:** MySQL 8
- **Contenedores:** Docker (una imagen por componente) + Docker Compose para desarrollo local
- **CI/CD:** GitHub Actions (build, test, push a Amazon ECR, deploy a EKS)
- **Orquestación en la nube:** Amazon EKS (Kubernetes gestionado), con Horizontal Pod Autoscaler
- **Observabilidad:** logs del pipeline en GitHub Actions + CloudWatch Container Insights (logs y métricas del clúster)

## Ejecutar en local (Docker Compose)

```bash
cp .env.example .env      # editar si se requiere
docker-compose up --build
```

- Frontend: http://localhost:8080
- Backend:  http://localhost:3001/api/productos
- MySQL:    localhost:3306

## Registro de imágenes (Amazon ECR)

Cada push a `main` construye y publica 3 imágenes, taggeadas con el SHA del commit (trazabilidad) y con
el tag `eks-v1` (última versión estable):

- `<ECR_URL>/tienda-backend`
- `<ECR_URL>/tienda-frontend`
- `<ECR_URL>/tienda-db`

## Pipeline de CI/CD (GitHub Actions)

Archivo: `.github/workflows/deploy.yml`. Se ejecuta en cada push a `main` y tiene 2 jobs:

1. **build-test-push:** instala dependencias del backend, corre un test (`npm test`, chequeo de sintaxis),
   construye las 3 imágenes Docker y las publica en Amazon ECR.
2. **deploy:** configura `kubectl` contra el clúster EKS, crea/actualiza el `Secret` de MySQL a partir de un
   GitHub Secret (nunca se sube la contraseña real al repo), reemplaza el placeholder `{{ECR_URL}}` en los
   manifiestos, y aplica los recursos de Kubernetes en orden (DB → backend → frontend), verificando el
   `rollout status` de cada Deployment.

### Secretos requeridos en el repositorio de GitHub

| Secret | Descripción |
|---|---|
| `AWS_ACCESS_KEY_ID` | Credencial de AWS Academy |
| `AWS_SECRET_ACCESS_KEY` | Credencial de AWS Academy |
| `AWS_SESSION_TOKEN` | Token de sesión (obligatorio en AWS Academy) |
| `DB_ROOT_PASSWORD` | Password root de MySQL, se inyecta como Kubernetes Secret en el deploy |

El rol de IAM utilizado en AWS Academy es `LabRole` (rol de laboratorio con permisos preconfigurados);
en un entorno productivo se recomienda reemplazarlo por un rol con permisos acotados exclusivamente a
ECR, EKS y CloudWatch (principio de mínimo privilegio).

## Despliegue manual (alternativa a CI/CD)

`deploy.sh` hace lo mismo que el pipeline pero desde la máquina local: actualiza `kubeconfig`, instala
Metrics Server y CloudWatch Container Insights, construye y publica las imágenes, y aplica los manifiestos
de `k8s/` en orden. Ver también `k8s/README.txt` para los pasos manuales con `kubectl` uno por uno.

```bash
chmod +x deploy.sh
./deploy.sh
```

## Kubernetes (carpeta `k8s/`)

| Archivo | Función |
|---|---|
| `namespace.yaml` | Namespace `tienda` |
| `mysql-deployment.yaml` / `mysql-service.yaml` / `mysql-secret.yaml` | Base de datos (Service headless, referencia de Secret para uso local) |
| `backend-deployment.yaml` / `backend-service.yaml` / `backend-hpa.yaml` | API (probes de liveness/readiness, resources, autoescalado 2-10 réplicas) |
| `frontend-deployment.yaml` / `frontend-service.yaml` / `frontend-hpa.yaml` | Frontend (Service tipo LoadBalancer, autoescalado 2-6 réplicas) |

## Observabilidad

- **Logs del pipeline:** disponibles en la pestaña *Actions* de GitHub, por cada job y step.
- **Logs y métricas del clúster:** CloudWatch Container Insights (namespace `amazon-cloudwatch`),
  instalado por `deploy.sh`. Se revisan desde la consola de AWS → CloudWatch → Container Insights,
  o vía `kubectl logs` sobre los pods de la aplicación.
- **Métricas para autoescalado:** Metrics Server + HPA (`kubectl get hpa -n tienda`).

## Seguridad básica aplicada

- Imágenes base minimalistas (`node:18-alpine`, `nginx:alpine`)
- `.dockerignore` en cada componente para no filtrar archivos innecesarios a la imagen
- Contraseña de MySQL gestionada como `Secret` de Kubernetes, generado en CI desde GitHub Secrets
  (nunca queda hardcodeada en el historial de commits del pipeline)
- Backend expone solo el puerto necesario (3001) y no se expone directamente a internet: el único
  Service público es el frontend (Nginx actúa como proxy hacia `/api/*`)
