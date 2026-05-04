# Manual de Despliegue y Operación — SecureVault

## 1. Prerrequisitos del Servidor

| Requisito | Versión mínima |
|-----------|----------------|
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
| `FERNET_KEY` | Clave de cifrado Fernet | Ver comando de generación más abajo |
| `RABBITMQ_URL` | URL de conexión RabbitMQ | `amqp://user:pass@host:5672/` |
| `ADMIN_USERNAME` | Nombre de usuario del administrador inicial | `admin` |
| `ADMIN_EMAIL` | Correo del administrador inicial | `admin@tudominio.com` |
| `ADMIN_PASSWORD` | Contraseña del administrador inicial | Mínimo 8 caracteres |
| `DOCKERHUB_USERNAME` | Usuario de Docker Hub (GitHub Secret para CI/CD) | `miusuario` |
| `DOCKERHUB_TOKEN` | Token de acceso Docker Hub (GitHub Secret para CI/CD) | `dckr_pat_...` |

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
```

Ejemplo de `.env` para producción:

```env
DATABASE_URL=postgresql://securevault:TU_DB_PASS@postgres:5432/securevault
SECRET_KEY=<resultado de openssl rand -hex 32>
FERNET_KEY=<resultado del comando python3>
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
ADMIN_USERNAME=admin
ADMIN_EMAIL=admin@tudominio.com
ADMIN_PASSWORD=TuPasswordSegura123!
DEBUG=false
ALLOW_SELF_REGISTER_ADMIN=false
SECRET_ROTATION_DAYS=90
```

```bash
# 3. Desplegar
docker compose up -d --build

# 4. Verificar estado
docker compose ps
curl http://localhost:8000/health
```

### Usuario administrador inicial

> ✅ **No es necesario crear el administrador manualmente.** Al arrancar, el API Gateway siembra automáticamente un usuario administrador con las credenciales definidas en las variables `ADMIN_USERNAME`, `ADMIN_EMAIL` y `ADMIN_PASSWORD` del archivo `.env`.

Para verificar que el admin fue creado correctamente, haz login directamente:

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "admin",
    "password": "TuPasswordSegura123!"
  }'
```

Respuesta esperada:

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "bearer",
  "role": "admin",
  "username": "admin"
}
```

> ⚠️ Cambia la contraseña del administrador desde la interfaz web (`/change-password`) inmediatamente después del primer despliegue.

> ⚠️ El auto-registro con rol `admin` está bloqueado por defecto (`ALLOW_SELF_REGISTER_ADMIN=false`). No intentes crear el admin vía el endpoint de registro — recibirás un `403 Forbidden`. El único mecanismo para el admin inicial es a través de las variables de entorno descritas arriba.

---

## 4. Despliegue con Ansible

```bash
# Instalar Ansible
pip install ansible

# Ejecutar playbook (desde la raíz del proyecto)
ansible-playbook infrastructure/ansible/site.yml
```

> El playbook usa la plantilla `infrastructure/ansible/templates/env.j2` para generar el archivo `.env` en el servidor destino. Asegúrate de definir las variables en el inventario (`infrastructure/ansible/inventories/production.ini`) antes de ejecutar.

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

> Las imágenes Docker deben estar publicadas en Docker Hub (o el registry configurado) antes de ejecutar Terraform. El pipeline de CI/CD en `.github/workflows/deploy.yml` se encarga de construir y subir las imágenes usando `DOCKERHUB_USERNAME` y `DOCKERHUB_TOKEN`.

---

## 6. Stack de Monitoreo (opcional)

```bash
# Levantar Prometheus + Grafana + Loki junto con los servicios principales
docker compose -f docker-compose.yml -f docker-compose.monitoring.yml up -d

# Acceder a Grafana
open http://localhost:3001
# Usuario: admin | Contraseña: admin
```

> ⚠️ Verifica el puerto de Grafana en `docker-compose.monitoring.yml` antes de acceder. Si el puerto `3001` está en conflicto con otro servicio en tu servidor, ajústalo en ese archivo.

---

## 7. Verificación del Despliegue

```bash
# Health check API
curl http://localhost:8000/health
# Esperado: {"status":"healthy","service":"api-gateway"}

# Verificar frontend
curl -I http://localhost:3000
# Esperado: HTTP/1.1 200 OK

# Verificar RabbitMQ Management UI
curl http://localhost:15672
# Credenciales por defecto: guest / guest

# Ver logs del worker de auditoría
docker compose logs worker-audit --tail=20

# Ver logs del API Gateway
docker compose logs api-gateway --tail=20
```

---

## 8. Troubleshooting Común

| Problema | Causa probable | Solución |
|----------|----------------|----------|
| `api-gateway` no inicia | PostgreSQL aún no está listo | Esperar el healthcheck; reiniciar con `docker compose restart api-gateway` |
| Worker no conecta a RabbitMQ | RabbitMQ tardó en iniciar | El worker tiene reintentos automáticos; esperar 30s y revisar logs |
| Error `FERNET_KEY invalid` | Clave mal formada | Regenerar con el comando Python indicado en la sección 2 |
| Puerto 5432 ya en uso | PostgreSQL local corriendo | Cambiar puerto en docker-compose: `"5433:5432"` |
| Frontend muestra error de CORS | `CORS_ORIGINS` mal configurado | Agregar la URL del frontend a la variable `CORS_ORIGINS` en el API |
| El admin inicial no existe | Variables `ADMIN_*` no definidas en `.env` | Verificar que `ADMIN_USERNAME`, `ADMIN_EMAIL` y `ADMIN_PASSWORD` estén en `.env` y reiniciar con `docker compose restart api-gateway` |
| `403 Forbidden` al intentar crear admin vía `/register` | `ALLOW_SELF_REGISTER_ADMIN=false` | El admin se crea automáticamente al arrancar. No usar el endpoint de registro para el rol admin |

---

## 9. Backup de Base de Datos

```bash
# Backup
docker exec sv_postgres pg_dump -U securevault securevault > backup_$(date +%Y%m%d).sql

# Restaurar
cat backup_20240101.sql | docker exec -i sv_postgres psql -U securevault securevault
```

> Automatiza los backups con un cron job en el servidor host para garantizar recuperación ante fallos.
