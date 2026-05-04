# Manual de Despliegue y Operación — SecureVault

## 1. Prerrequisitos del Servidor

| Requisito | Versión mínima |
|-----------|---------------|
| OS | Ubuntu 22.04 / Debian 12 / cualquier Linux con Docker |
| Docker | 24.0+ |
| Docker Compose | 2.20+ |
| CPU | 2 vCPUs |
| RAM | 2 GB |
| Disco | 20 GB |
| Puertos abiertos | 3000, 8000 |

---

## 2. Variables de Entorno en Producción

> ⚠️ **Nunca commitear valores reales al repositorio.** Usar GitHub Secrets para el pipeline y `.env` local para desarrollo.

### Variables obligatorias

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `DATABASE_URL` | Cadena de conexión PostgreSQL | `postgresql://user:pass@host:5432/db` |
| `SECRET_KEY` | Clave de firma JWT — mínimo 32 caracteres aleatorios | `openssl rand -hex 32` |
| `FERNET_KEY` | Clave de cifrado Fernet | `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"` |
| `RABBITMQ_URL` | URL de conexión RabbitMQ | `amqp://user:pass@host:5672/` |
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub (GitHub Secret) | `miusuario` |
| `DOCKERHUB_TOKEN` | Token de acceso Docker Hub (GitHub Secret) | `dckr_pat_...` |

### Generar claves seguras
```bash
# SECRET_KEY
openssl rand -hex 32

# FERNET_KEY
python3 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
```

---

## 3. Despliegue con Docker Compose (recomendado)

```bash
# 1. Clonar repositorio
git clone https://github.com/TU_USUARIO/securevault.git
cd securevault

# 2. Configurar variables de entorno
cp .env.example .env
nano .env   # Editar con valores de producción

# 3. Desplegar
docker compose up -d --build

# 4. Verificar estado
docker compose ps
curl http://localhost:8000/health

# 5. Crear usuario administrador inicial
curl -X POST http://localhost:8000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "email": "admin@tudominio.com",
    "password": "TuPasswordSegura123!",
    "role": "admin"
  }'
```

---

## 4. Despliegue con Ansible

```bash
# Instalar Ansible
pip install ansible

# Ejecutar playbook (desde la raíz del proyecto)
ansible-playbook infrastructure/ansible/site.yml
```

---

## 5. Despliegue con Terraform

```bash
cd infrastructure/terraform

# Inicializar Terraform
terraform init

# Revisar plan
terraform plan -var="db_password=tupassword" \
               -var="secret_key=$(openssl rand -hex 32)" \
               -var="fernet_key=TU_FERNET_KEY" \
               -var="docker_registry=tuusuario"

# Aplicar
terraform apply
```

---

## 6. Stack de Monitoreo (opcional)

```bash
# Levantar Prometheus + Grafana + Loki
docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d

# Acceder a Grafana
open http://localhost:3001
# Usuario: admin | Contraseña: admin
```

---

## 7. Verificación del Despliegue

```bash
# Health check API
curl http://localhost:8000/health
# Esperado: {"status":"healthy","service":"api-gateway"}

# Verificar frontend
curl -I http://localhost:3000
# Esperado: HTTP/1.1 200 OK

# Verificar RabbitMQ
curl http://localhost:15672  # Management UI

# Ver logs del worker
docker compose logs worker-audit --tail=20
```

---

## 8. Troubleshooting Común

| Problema | Causa probable | Solución |
|---------|----------------|----------|
| `api-gateway` no inicia | PostgreSQL aún no está listo | Esperar el healthcheck; reiniciar con `docker compose restart api-gateway` |
| Worker no conecta a RabbitMQ | RabbitMQ tardó en iniciar | El worker tiene reintentos automáticos; esperar 30s y revisar logs |
| Error `FERNET_KEY invalid` | Clave mal formada | Regenerar con el comando de Python indicado arriba |
| Puerto 5432 ya en uso | PostgreSQL local corriendo | Cambiar puerto en docker-compose: `"5433:5432"` |
| Frontend muestra error de CORS | `CORS_ORIGINS` mal configurado | Agregar la URL del frontend a la variable `CORS_ORIGINS` en el API |

---

## 9. Backup de Base de Datos

```bash
# Backup
docker exec sv_postgres pg_dump -U securevault securevault > backup_$(date +%Y%m%d).sql

# Restaurar
cat backup_20240101.sql | docker exec -i sv_postgres psql -U securevault securevault
```
