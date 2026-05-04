# Manual de Desarrollo — SecureVault

## 1. Configurar el Entorno Local

### Prerrequisitos
```
Docker >= 24.0
Docker Compose >= 2.20
Python 3.11+
Node.js 20+
Git
```

### Clonar y configurar
```bash
git clone https://github.com/TU_USUARIO/securevault.git
cd securevault

# Copiar archivo de variables de entorno
cp .env.example .env
# Editar .env con tus valores locales (para desarrollo los defaults funcionan)
```

### Archivo `.env.example`
```env
DATABASE_URL=postgresql://securevault:securevault@postgres:5432/securevault
RABBITMQ_URL=amqp://guest:guest@rabbitmq:5672/
SECRET_KEY=dev-secret-key-change-in-production-123456789
FERNET_KEY=TlgmqrgYgDKS7jKdLdZEJn-4RzLMKqy0LqJKJLWqoaA=
DEBUG=true
SECRET_ROTATION_DAYS=90
```

---

## 2. Modo Desarrollo con Docker Compose

```bash
# Construir e iniciar todos los servicios
docker compose up -d --build

# Ver logs en tiempo real
docker compose logs -f

# Ver logs de un servicio específico
docker compose logs -f api-gateway
docker compose logs -f worker-audit

# Reiniciar un servicio tras cambios
docker compose restart api-gateway

# Detener todo
docker compose down

# Detener y eliminar volúmenes (reset total)
docker compose down -v
```

---

## 3. Desarrollo Local sin Docker (solo backend)

```bash
# Crear entorno virtual
cd api-gateway
python -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate

# Instalar dependencias
pip install -r requirements.txt
pip install pytest pytest-cov httpx

# Apuntar a postgres local o usar SQLite para tests
export DATABASE_URL="sqlite:///./dev.db"
export SECRET_KEY="local-dev-key"
export FERNET_KEY="TlgmqrgYgDKS7jKdLdZEJn-4RzLMKqy0LqJKJLWqoaA="

# Iniciar servidor de desarrollo
uvicorn app.main:app --reload --port 8000
```

---

## 4. Ejecutar Pruebas

### Backend (Pytest)
```bash
cd api-gateway

# Ejecutar todos los tests
pytest tests/ -v

# Con reporte de cobertura
pytest tests/ --cov=app --cov-report=html --cov-report=term-missing

# Abrir reporte HTML
open htmlcov/index.html
```

### Frontend (Jest)
```bash
cd frontend

npm install
npm test -- --watchAll=false --coverage

# Ver reporte
open coverage/lcov-report/index.html
```

---

## 5. Estrategia de Branching

```
main          ← producción estable; protegida
  └── develop ← integración
        ├── feature/nombre-feature
        ├── fix/nombre-bug
        └── security/descripcion-fix
```

**Reglas:**
- Nunca hacer push directo a `main`
- Todo cambio a `main` requiere PR con al menos 1 aprobación
- El pipeline CI debe pasar antes de hacer merge

---

## 6. Convención de Commits

```
tipo(scope): descripción corta en presente

Tipos válidos:
  feat     - nueva funcionalidad
  fix      - corrección de bug
  security - parche de seguridad
  docs     - documentación
  test     - pruebas
  ci       - cambios en pipeline
  refactor - refactorización sin cambio funcional
  chore    - tareas de mantenimiento

Ejemplos:
  feat(secrets): agregar endpoint de rotación de secretos
  fix(auth): corregir validación de refresh token vencido
  security(deps): actualizar cryptography a 42.0.8
  ci(pipeline): agregar paso de Checkov para Terraform
```

---

## 7. Generar una nueva Fernet Key

```python
from cryptography.fernet import Fernet
print(Fernet.generate_key().decode())
```

> ⚠️ Si cambias la Fernet key en producción, todos los secretos cifrados quedan ilegibles. Implementa re-cifrado antes de rotar la clave.
