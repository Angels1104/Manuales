# Manual de Usuario — SecureVault

## 1. ¿Qué es SecureVault?

SecureVault es una plataforma web para almacenar, gestionar y auditar credenciales sensibles (contraseñas, API keys, tokens) de forma segura. Todos los valores se cifran antes de guardarse. Solo tú —o un administrador— puedes ver tus secretos.

---

## 2. Registro e Inicio de Sesión

### Registrarse
1. Ve a `http://localhost:3000/register`
2. Ingresa: **username**, **email**, **password** y elige un **rol**
3. Haz clic en **REGISTER**
4. Serás redirigido al login

### Roles disponibles
| Rol | Puede crear secretos | Puede rotar/eliminar | Ve todos los secretos |
|-----|---------------------|---------------------|-----------------------|
| **admin** | ✅ | ✅ | ✅ (todos los usuarios) |
| **editor** | ✅ | ✅ (solo los suyos) | ❌ (solo los suyos) |
| **viewer** | ❌ | ❌ | ✅ (solo los suyos, sin crear) |

### Iniciar sesión
1. Ve a `http://localhost:3000/login`
2. Ingresa tu **username** y **password**
3. Haz clic en **LOGIN**

---

## 3. Dashboard

Al iniciar sesión verás el panel principal con:

- **Total Secrets**: cuántos secretos tienes almacenados
- **Expired Secrets**: secretos que no han sido rotados en 90+ días ⚠️
- **Audit Events**: total de acciones registradas
- **Recent Activity**: últimas 5 acciones realizadas en tu cuenta

> Si hay secretos expirados, aparecerá una alerta roja con un enlace a la página de Secrets.

---

## 4. Gestión de Secretos

Ve a **Secrets** en el menú lateral.

### Crear un secreto
1. Haz clic en **+ New Secret**
2. Completa el formulario:
   - **Name**: identificador del secreto (ej. `AWS_ACCESS_KEY`)
   - **Category**: tipo de secreto (`api_key`, `password`, `token`, `certificate`, `other`)
   - **Secret Value**: el valor sensible (se cifrará automáticamente)
   - **Description**: descripción opcional
3. Haz clic en **STORE SECRET**

> Solo los roles **editor** y **admin** pueden crear secretos.

### Ver (revelar) el valor de un secreto
1. En la tabla de secretos, haz clic en **reveal** en la fila del secreto
2. El valor descifrado aparecerá en color amarillo
3. Haz clic en **hide** para ocultarlo nuevamente

> Cada reveal queda registrado en el Audit Log.

### Rotar un secreto
Cuando un secreto cambia (por ejemplo, regeneraste una API key):
1. Haz clic en **rotate** en la fila del secreto
2. Ingresa el **nuevo valor** en el campo que aparece
3. Haz clic en **Confirm Rotate**
4. El estado del secreto vuelve a `active` y la fecha de rotación se actualiza

### Eliminar un secreto
1. Haz clic en **delete** en la fila del secreto
2. Confirma en el diálogo de confirmación

> ⚠️ Esta acción es irreversible.

### Estados de un secreto
| Estado | Significado |
|--------|-------------|
| `active` | El secreto está vigente y fue rotado recientemente |
| `expired` | No se ha rotado en 90+ días — el worker lo marcó automáticamente |
| `rotated` | Ha sido actualizado al menos una vez |

---

## 5. Audit Log

Ve a **Audit Log** en el menú lateral.

Muestra un registro inmutable de todas las acciones realizadas:

| Acción | Descripción |
|--------|-------------|
| `REGISTER` | Se registró un nuevo usuario |
| `LOGIN` | Inicio de sesión exitoso |
| `CREATE_SECRET` | Se almacenó un nuevo secreto |
| `REVEAL_SECRET` | Se descifró y visualizó un secreto |
| `ROTATE_SECRET` | Se actualizó el valor de un secreto |
| `DELETE_SECRET` | Se eliminó un secreto |

### Filtrar eventos
Escribe en el campo de búsqueda para filtrar por tipo de acción (ej. escribe `REVEAL` para ver solo los accesos a secretos).

> Los **admins** ven todos los eventos de todos los usuarios. Los **editors** y **viewers** solo ven los suyos.

---

## 6. Cerrar Sesión

Haz clic en el botón ⏻ **Logout** en la parte inferior del menú lateral.

---

## 7. Preguntas Frecuentes

**¿Qué pasa si olvido mi contraseña?**
En v1.0 no hay recuperación automática de contraseña. Un admin puede eliminar y recrear tu cuenta.

**¿Mis secretos están seguros?**
Sí. Los valores se cifran con **AES-128** (Fernet) antes de guardarse en la base de datos. Ni los administradores del sistema pueden ver los valores sin descifrarlos explícitamente a través de la aplicación, lo cual queda registrado en el audit log.

**¿Por qué aparece mi secreto como "expired"?**
El worker de auditoría revisa periódicamente los secretos y marca como `expired` aquellos que no han sido rotados en más de 90 días. Rota el secreto para que vuelva a estado `active`.

**¿Puedo ver los secretos de otro usuario?**
Solo si eres **admin**. Los roles editor y viewer solo pueden ver y gestionar sus propios secretos.
