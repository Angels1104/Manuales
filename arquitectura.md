# Manual de Arquitectura — SecureVault

## 1. Descripción General

SecureVault implementa una **arquitectura de microservicios** donde cada componente tiene una responsabilidad única y se comunica a través de interfaces bien definidas: HTTP/REST para comunicación sincrónica y AMQP (RabbitMQ) para comunicación asíncrona.

### Principios de diseño
- **Separación de responsabilidades**: cada servicio hace una sola cosa bien
- **Comunicación asíncrona**: el worker no bloquea al API gateway
- **Cifrado en reposo**: los valores de los secretos se cifran con Fernet (AES-128-CBC) antes de persistirse
- **Principio de mínimo privilegio**: contenedores corren como usuario no-root; roles granulares en la API

---

## 2. Diagrama de Componentes

```mermaid
graph TB
    User([Usuario])

    subgraph "Frontend — React SPA :3000"
        FE[React + Vite + Tailwind]
    end

    subgraph "API Gateway — FastAPI :8000"
        AUTH[/auth router/]
        SEC[/secrets router/]
        AUD[/audit router/]
        JWT_MW[JWT Middleware]
        ENC[Fernet Encryption]
    end

    subgraph "Broker"
        RMQ[(RabbitMQ\nqueue: secret_events\nqueue: secret_alerts)]
    end

    subgraph "Worker Audit — Python"
        W_LISTEN[Event Listener]
        W_CHECK[Expiry Checker\ncron cada 30s]
    end

    subgraph "Persistencia"
        PG[(PostgreSQL\ntablas: users\nsecrets\naudit_logs)]
    end

    User --> FE
    FE --> JWT_MW
    JWT_MW --> AUTH
    JWT_MW --> SEC
    JWT_MW --> AUD
    SEC --> ENC
    ENC --> PG
    AUTH --> PG
    AUD --> PG
    SEC -->|AMQP publish| RMQ
    RMQ -->|consume| W_LISTEN
    W_CHECK -->|UPDATE secrets| PG
    W_CHECK -->|publish alert| RMQ
```

---

## 3. Diagrama de Despliegue

```mermaid
graph LR
    subgraph "Docker Network: sv_network"
        FE_C[Container: sv_frontend\nNginx :3000]
        API_C[Container: sv_api\nuvicorn :8000]
        WRK_C[Container: sv_worker\nPython]
        RMQ_C[Container: sv_rabbitmq\n:5672 / :15672]
        PG_C[Container: sv_postgres\n:5432]
    end

    subgraph "Volúmenes"
        VOL[(postgres_data)]
    end

    subgraph "Host"
        HOST_3000[":3000"]
        HOST_8000[":8000"]
        HOST_15672[":15672"]
    end

    HOST_3000 --> FE_C
    HOST_8000 --> API_C
    HOST_15672 --> RMQ_C
    FE_C --> API_C
    API_C --> PG_C
    API_C --> RMQ_C
    WRK_C --> PG_C
    WRK_C --> RMQ_C
    PG_C --- VOL
```

---

## 4. Diagrama de Secuencia — Autenticación JWT

```mermaid
sequenceDiagram
    actor U as Usuario
    participant FE as Frontend (React)
    participant GW as API Gateway (FastAPI)
    participant DB as PostgreSQL

    U->>FE: Ingresa username + password
    FE->>GW: POST /api/v1/auth/login
    GW->>DB: SELECT user WHERE username=?
    DB-->>GW: User record + hashed_password

    alt Credenciales válidas
        GW->>GW: bcrypt.verify(password, hash)
        GW->>GW: Genera JWT (30min) + Refresh Token (7d)
        GW->>DB: INSERT audit_log (LOGIN)
        GW-->>FE: 200 {access_token, refresh_token, role}
        FE->>FE: Guarda tokens en localStorage
        FE-->>U: Redirige a /dashboard
    else Credenciales inválidas
        GW-->>FE: 401 Unauthorized
        FE-->>U: Muestra error
    end

    Note over FE,GW: Requests subsecuentes incluyen\nAuthorization: Bearer <access_token>
```

---

## 5. Diagrama de Secuencia — Almacenar y Rotar Secreto

```mermaid
sequenceDiagram
    actor U as Usuario (editor/admin)
    participant FE as Frontend
    participant GW as API Gateway
    participant ENC as Fernet Cipher
    participant DB as PostgreSQL
    participant RMQ as RabbitMQ
    participant WRK as Worker Audit

    U->>FE: Formulario: nombre + valor + categoría
    FE->>GW: POST /api/v1/secrets/ + JWT
    GW->>GW: Valida JWT + rol (viewer bloqueado)
    GW->>ENC: encrypt(valor)
    ENC-->>GW: valor_cifrado (AES-128)
    GW->>DB: INSERT secret (nombre, cifrado, owner_id)
    GW->>DB: INSERT audit_log (CREATE_SECRET)
    GW->>RMQ: publish → secret_events
    GW-->>FE: 201 {id, nombre, categoría, status}
    FE-->>U: Secreto creado

    RMQ-->>WRK: consume secret_events
    WRK->>WRK: log evento recibido

    Note over WRK,DB: Cada 30 segundos
    WRK->>DB: UPDATE secrets SET status='expired'\nWHERE last_rotated_at < NOW()-90d
    WRK->>RMQ: publish → secret_alerts (si hay expirados)
```

---

## 6. Diagrama de Casos de Uso

```mermaid
flowchart LR
    Admin([Admin])
    Editor([Editor])
    Viewer([Viewer])
    Worker([Worker\nAudit])

    subgraph "SecureVault"
        UC1[Registrar usuario]
        UC2[Iniciar sesión]
        UC3[Crear secreto]
        UC4[Listar secretos]
        UC5[Revelar secreto]
        UC6[Rotar secreto]
        UC7[Eliminar secreto]
        UC8[Ver audit log completo]
        UC9[Ver mi audit log]
        UC10[Ver estadísticas]
        UC11[Marcar secretos expirados]
        UC12[Publicar alertas de expiración]
    end

    Admin --> UC1
    Admin --> UC2
    Admin --> UC3
    Admin --> UC4
    Admin --> UC5
    Admin --> UC6
    Admin --> UC7
    Admin --> UC8
    Admin --> UC10

    Editor --> UC2
    Editor --> UC3
    Editor --> UC4
    Editor --> UC5
    Editor --> UC6
    Editor --> UC9

    Viewer --> UC2
    Viewer --> UC4
    Viewer --> UC5
    Viewer --> UC9

    Worker --> UC11
    Worker --> UC12
```

---

## 7. DFD Nivel 0 — Vista General

```mermaid
graph LR
    U([Usuario])
    SV[[SecureVault\nSistema]]
    DB[(PostgreSQL)]
    RMQ[(RabbitMQ)]

    U -->|credenciales / secretos| SV
    SV -->|tokens JWT / secretos cifrados| U
    SV <-->|CRUD cifrado| DB
    SV <-->|eventos async| RMQ
```

---

## 8. DFD Nivel 1 — Flujos Internos

```mermaid
graph TB
    U([Usuario])

    P1[1.0\nAutenticación\nJWT]
    P2[2.0\nGestión de\nSecrets]
    P3[3.0\nAudit\nLogger]
    P4[4.0\nWorker\nAudit]

    DS1[(users)]
    DS2[(secrets\ncifrados)]
    DS3[(audit_logs)]
    RMQ[(RabbitMQ)]

    U -->|username+pass| P1
    P1 -->|JWT token| U
    P1 -->|read/write| DS1
    P1 -->|audit event| P3

    U -->|JWT + datos secreto| P2
    P2 -->|secreto cifrado| DS2
    P2 -->|audit event| P3
    P2 -->|secret_event| RMQ

    P3 -->|INSERT| DS3

    RMQ -->|consume| P4
    P4 -->|UPDATE expired| DS2
    P4 -->|alert event| RMQ
```

---

## 9. Modelo de Datos

```mermaid
erDiagram
    users {
        UUID id PK
        VARCHAR username
        VARCHAR email
        VARCHAR hashed_password
        ENUM role
        BOOLEAN is_active
        TIMESTAMP created_at
    }

    secrets {
        UUID id PK
        VARCHAR name
        TEXT description
        TEXT encrypted_value
        ENUM category
        ENUM status
        UUID owner_id FK
        TIMESTAMP created_at
        TIMESTAMP updated_at
        TIMESTAMP last_rotated_at
    }

    audit_logs {
        UUID id PK
        UUID user_id FK
        VARCHAR action
        VARCHAR resource
        VARCHAR resource_id
        TEXT details
        VARCHAR ip_address
        TIMESTAMP timestamp
    }

    users ||--o{ secrets : "owns"
    users ||--o{ audit_logs : "generates"
```
