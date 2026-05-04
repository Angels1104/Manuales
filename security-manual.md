# Manual de Seguridad — SecureVault

## 1. Modelo de Amenazas (STRIDE)

El modelado de amenazas se realizó con **OWASP Threat Dragon** sobre los flujos críticos de la aplicación.

### 1.1 Categorías STRIDE Analizadas

| Categoría | Amenaza Identificada | Componente Afectado | Severidad | Contramedida Implementada | Estado |
|-----------|---------------------|-------------------|-----------|--------------------------|--------|
| **S**poofing | Suplantación de identidad de usuario mediante credenciales robadas | API Gateway / Auth | Alta | JWT con expiración corta (30min) + Refresh Token rotado (7d) + bcrypt con salt | ✅ Mitigado |
| **S**poofing | Uso de token JWT vencido o manipulado | API Gateway | Alta | Verificación de firma y expiración en cada request; clave secreta nunca expuesta | ✅ Mitigado |
| **T**ampering | Modificación del valor de un secreto en tránsito | Red interna | Media | HTTPS en producción; cifrado Fernet antes de escribir en DB | ✅ Mitigado |
| **T**ampering | Modificación directa de la base de datos | PostgreSQL | Alta | Credenciales DB solo accesibles vía variables de entorno; usuario DB sin acceso externo | ✅ Mitigado |
| **R**epudiation | Negar haber revelado o eliminado un secreto | Todos | Media | Audit log inmutable (INSERT only, sin UPDATE/DELETE en audit_logs) | ✅ Mitigado |
| **I**nformation Disclosure | Filtración de secretos en texto plano en logs | API Gateway | Crítica | Los valores cifrados nunca se loguean; Gitleaks en pre-commit detecta secretos en código | ✅ Mitigado |
| **I**nformation Disclosure | Secretos expuestos en variables de entorno del repositorio | CI/CD | Crítica | GitHub Secrets para todas las variables sensibles; `.env` en `.gitignore` | ✅ Mitigado |
| **I**nformation Disclosure | Usuario visualiza secretos de otro usuario | API Gateway | Alta | Validación de `owner_id` en cada operación; solo admin puede ver todos | ✅ Mitigado |
| **D**enial of Service | Flood de requests al endpoint de login | API Gateway | Media | Recomendado: rate limiting con `slowapi` (pendiente en v1.1) | ⚠️ Aceptado |
| **D**enial of Service | Creación masiva de secretos por un usuario | API Gateway | Baja | Paginación en listados; sin límite implementado en v1.0 | ⚠️ Aceptado |
| **E**levation of Privilege | Viewer intenta crear o rotar secretos | API Gateway | Alta | Validación explícita de rol en cada endpoint sensible; 403 Forbidden | ✅ Mitigado |
| **E**levation of Privilege | Escalada horizontal: editor accede a secretos de otro usuario | API Gateway | Alta | Filtro por `owner_id` en todas las queries; admin es el único con vista global | ✅ Mitigado |

### 1.2 Riesgos Residuales Aceptados

- **Rate limiting**: No implementado en v1.0. Riesgo aceptado dado que la aplicación es para equipos internos pequeños. Se planifica `slowapi` en v1.1.
- **MFA (Multi-Factor Authentication)**: No implementado. Fuera del alcance de v1.0.
- **Rotación automática de Fernet key**: Manual en v1.0. Re-cifrado de secretos ante rotación de clave es trabajo futuro.

---

## 2. Herramientas de Seguridad Integradas

### 2.1 Gitleaks — Detección de Secretos en Código

**¿Qué hace?** Escanea el historial completo de git en busca de secretos (API keys, contraseñas, tokens) expuestos en el código fuente.

**Configuración en el pipeline:**
```yaml
- uses: gitleaks/gitleaks-action@v2
  with:
    fetch-depth: 0   # Escanea TODO el historial
```

**Interpretar el reporte:**
- Si Gitleaks encuentra un secreto, el pipeline falla con código de salida 1.
- El reporte indica el archivo, línea y tipo de secreto encontrado.
- **Acción requerida**: rotar el secreto inmediatamente, removerlo del código y del historial git (`git filter-branch` o `git-filter-repo`).

---

### 2.2 Bandit — SAST Python

**¿Qué hace?** Analiza el código Python en busca de patrones de seguridad problemáticos (inyección SQL, uso inseguro de `eval`, algoritmos criptográficos débiles, etc.).

**Ejecutar localmente:**
```bash
pip install bandit
bandit -r api-gateway/app/ -ll -f json -o bandit-report.json
```

**Interpretar niveles de severidad:**
| Nivel | Acción |
|-------|--------|
| LOW | Revisar; probablemente aceptable |
| MEDIUM | Evaluar y documentar si se acepta |
| HIGH | Corregir antes de merge a main |

**Falsos positivos comunes en este proyecto:**
- `B105` (hardcoded password): Los valores en `config.py` son defaults de desarrollo, no credenciales reales. Se justifican con comentario `# nosec B105`.

---

### 2.3 Trivy — Escaneo de Imágenes Docker y Dependencias

**¿Qué hace?** Detecta CVEs (vulnerabilidades conocidas) en:
- Dependencias de Python (`requirements.txt`)
- Paquetes npm (`package.json`)
- Imágenes Docker (OS base + paquetes instalados)

**El pipeline falla automáticamente si hay CVEs CRÍTICOS** sin excepción documentada.

**Ejecutar localmente:**
```bash
# Escanear imagen
trivy image securevault/api-gateway:latest

# Escanear dependencias
trivy fs api-gateway/requirements.txt --severity HIGH,CRITICAL
```

**Interpretar el reporte:**
```
┌────────────┬───────────────┬──────────┬──────────────────┐
│  Library   │ Vulnerability │ Severity │    Fix Version   │
├────────────┼───────────────┼──────────┼──────────────────┤
│ cryptography│ CVE-XXXX-YYYY│ CRITICAL │     42.0.8       │
└────────────┴───────────────┴──────────┴──────────────────┘
```

**Proceso de gestión**: actualizar la dependencia a la versión corregida, re-ejecutar el pipeline.

---

### 2.4 OWASP ZAP — DAST

**¿Qué hace?** Realiza un escaneo dinámico de la aplicación en ejecución, simulando ataques reales (XSS, SQL injection, CSRF, headers inseguros, etc.).

**Modo usado:** Baseline Scan (pasivo + activo básico) contra `http://localhost:8000`.

**Interpretar alertas:**
| Riesgo | Acción |
|--------|--------|
| High | Corregir antes de release |
| Medium | Evaluar e incluir en backlog priorizado |
| Low / Informational | Documentar; corregir si es bajo costo |

**Alertas esperadas en este proyecto (informacionales):**
- Missing `Content-Security-Policy` header → agregar en nginx en producción.
- `X-Frame-Options` → ya configurado en nginx.conf.

---

### 2.5 Checkov — Escaneo de IaC

**¿Qué hace?** Analiza archivos Terraform y Docker Compose en busca de malas configuraciones de seguridad (contenedores con privilegios, secrets en texto plano, redes inseguras, etc.).

**Ejecutar localmente:**
```bash
pip install checkov
checkov -d infrastructure/terraform --framework terraform
checkov -f docker-compose.yml --framework dockerfile
```

---

## 3. Política de Divulgación Responsable

Si encuentras una vulnerabilidad de seguridad en SecureVault:

1. **No la publiques públicamente** hasta que sea corregida.
2. Envía un reporte detallado a: `security@securevault.local` (o abre un GitHub Issue privado).
3. Incluye: descripción, pasos para reproducir, impacto estimado y posible solución.
4. El equipo responderá en máximo **72 horas** con un plan de acción.
5. Una vez corregida, se publicará un `SECURITY.md` con el CVE asignado y crédito al reportante.

---

## 4. Gestión de Vulnerabilidades — Tabla de Hallazgos

| Herramienta | Hallazgo | Severidad | Estado | Justificación |
|-------------|---------|-----------|--------|---------------|
| Bandit | B104: Binding to all interfaces (0.0.0.0) | Low | Aceptado | Necesario para Docker; no expuesto directamente a internet |
| Trivy (imagen) | python:3.11-slim CVEs de SO | Medium | Monitorizado | Imagen base actualizada en cada build; sin CVEs críticos |
| ZAP | Missing CSP header | Medium | Aceptado v1.0 | Se añadirá en nginx de producción en v1.1 |
| Checkov | CKV_DOCKER_2: sin HEALTHCHECK en algunos Dockerfiles | Low | Corregido | HEALTHCHECK añadido via docker-compose |
| Gitleaks | Ninguno detectado | — | ✅ Limpio | — |
