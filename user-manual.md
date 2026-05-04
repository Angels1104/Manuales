# Manual de Usuario — SecureVault

## 1. ¿Qué es SecureVault?

SecureVault es una plataforma web para almacenar, gestionar y auditar credenciales sensibles (contraseñas, API keys, tokens) de forma segura. Todos los valores se cifran con **Fernet (AES-128-CBC)** antes de guardarse en la base de datos. Solo tú —o un administrador— puedes ver tus secretos, salvo que alguien con permisos te comparta acceso explícitamente.

---

## 2. Registro e Inicio de Sesión

### Registrarse

1. Ve a `http://localhost:3000/register`
2. Ingresa: **username**, **email**, **password** y elige un **rol**
3. Haz clic en **REGISTER**
4. Serás redirigido al login

> ⚠️ El rol **admin** no puede seleccionarse durante el auto-registro. El administrador inicial es creado automáticamente por el sistema al arrancar. Si necesitas una cuenta admin adicional, contacta al administrador del sistema.

### Roles disponibles

| Rol | Puede crear secretos | Puede rotar/editar/eliminar | Ve todos los secretos | Puede compartir secretos |
|-----|---------------------|-----------------------------|-----------------------|--------------------------|
| **admin** | ✅ | ✅ (todos) | ✅ (todos los usuarios) | ✅ (a cualquier usuario) |
| **editor** | ✅ | ✅ (solo los suyos o compartidos) | ❌ (solo los suyos y los compartidos) | ✅ (solo a viewers) |
| **viewer** | ❌ | ❌ | ✅ (solo los suyos y los que le compartan) | ❌ |

### Iniciar sesión

1. Ve a `http://localhost:3000/login`
2. Ingresa tu **username** y **password**
3. Haz clic en **LOGIN**

---

## 3. Dashboard

Al iniciar sesión verás el panel principal con:

- **Total Secrets**: cuántos secretos tienes almacenados
- **Expired Secrets**: secretos que no han sido rotados en 90+ días ⚠️
- **Audit Events**: total de acciones registradas en tu cuenta
- **Recent Activity**: últimas 5 acciones realizadas

> Si hay secretos expirados, aparecerá una alerta roja con un enlace a la página de Secrets.

---

## 4. Gestión de Secretos

Ve a **Secrets** en el menú lateral.

### Crear un secreto

1. Haz clic en **+ New Secret**
2. Completa el formulario:
   - **Name**: identificador del secreto (ej. `AWS_ACCESS_KEY`)
   - **Category**: tipo de secreto (`api_key`, `password`, `token`, `certificate`, `other`)
   - **Secret Value**: el valor sensible (se cifrará automáticamente con Fernet)
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

### Editar un secreto

Puedes actualizar el nombre, descripción o categoría de un secreto sin cambiar su valor:

1. Haz clic en **edit** en la fila del secreto
2. Modifica los campos deseados
3. Guarda los cambios

> Para cambiar el valor cifrado usa **rotate**, no edit.

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

## 5. Compartir Secretos

Los secretos pueden compartirse con otros usuarios para que puedan consultarlos sin ser el propietario.

### Compartir un secreto (admin o editor propietario)

1. En la tabla de secretos, abre el menú de accesos del secreto que deseas compartir
2. Selecciona el usuario al que quieres otorgar acceso
3. Confirma la acción

> Los **admins** pueden compartir secretos con cualquier usuario. Los **editors** solo pueden compartir sus propios secretos y únicamente con usuarios de rol **viewer**.

### Ver secretos compartidos contigo

Los secretos que otros usuarios te hayan compartido aparecen automáticamente en tu lista de secretos, marcados con la etiqueta **shared**.

### Revocar acceso

El propietario del secreto (o un admin) puede revocar el acceso a un usuario en cualquier momento desde el panel de gestión de accesos del secreto.

---

## 6. Cambiar Contraseña

Puedes cambiar tu contraseña en cualquier momento sin necesidad de contactar al administrador:

1. Ve a **Change Password** en el menú lateral (o navega a `http://localhost:3000/change-password`)
2. Ingresa tu **contraseña actual**
3. Ingresa la **nueva contraseña** (mínimo 8 caracteres)
4. Confirma y guarda

> La acción queda registrada en el Audit Log.

---

## 7. Audit Log

Ve a **Audit Log** en el menú lateral.

Muestra un registro de todas las acciones realizadas en el sistema:

| Acción | Descripción |
|--------|-------------|
| `REGISTER` | Se registró un nuevo usuario |
| `LOGIN` | Inicio de sesión exitoso |
| `CREATE_SECRET` | Se almacenó un nuevo secreto |
| `REVEAL_SECRET` | Se descifró y visualizó un secreto |
| `ROTATE_SECRET` | Se actualizó el valor de un secreto |
| `UPDATE_SECRET` | Se modificaron los metadatos de un secreto (nombre, descripción, categoría) |
| `DELETE_SECRET` | Se eliminó un secreto |
| `GRANT_ACCESS` | Se otorgó acceso a un secreto a otro usuario |
| `REVOKE_ACCESS` | Se revocó el acceso de un usuario a un secreto |
| `CHANGE_PASSWORD` | Un usuario cambió su contraseña |

### Filtrar eventos

Escribe en el campo de búsqueda para filtrar por tipo de acción (ej. escribe `REVEAL` para ver solo los accesos a secretos).

> Los **admins** ven todos los eventos de todos los usuarios. Los **editors** y **viewers** solo ven los suyos.

---

## 8. Cerrar Sesión

Haz clic en el botón ⏻ **Logout** en la parte inferior del menú lateral.

---

## 9. Preguntas Frecuentes

**¿Qué pasa si olvido mi contraseña?**
En v1.0 no hay recuperación automática de contraseña por correo. Sin embargo, si recuerdas tu contraseña actual puedes cambiarla desde la sección **Change Password**. Si la olvidaste por completo, un administrador puede eliminar y recrear tu cuenta.

**¿Mis secretos están seguros?**
Sí. Los valores se cifran con **Fernet (AES-128-CBC)** antes de guardarse en la base de datos. Ni los administradores del sistema pueden ver los valores sin descifrarlos explícitamente a través de la aplicación, lo cual queda registrado en el audit log.

**¿Por qué aparece mi secreto como "expired"?**
El worker de auditoría revisa periódicamente los secretos y marca como `expired` aquellos que no han sido rotados en más de 90 días. Rota el secreto para que vuelva a estado `active`.

**¿Puedo ver los secretos de otro usuario?**
Solo si eres **admin**, o si el propietario del secreto te ha otorgado acceso explícitamente. Los secretos compartidos contigo aparecen en tu lista marcados como **shared**.

**¿Puedo registrarme con rol admin?**
No. El auto-registro con rol admin está deshabilitado por seguridad. El administrador inicial es creado automáticamente por el sistema. Para cuentas admin adicionales, contacta al administrador del sistema.
