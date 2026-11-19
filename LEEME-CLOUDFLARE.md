# STORE MOLANO 1.1 — IMAP Gmail y múltiples buzones guardados en GitHub

## Qué se conserva

Los usuarios siguen en const USERS de script.js. Este proyecto no asigna listas de correos a usuarios; conserva su formato original. Se conservan los seis usuarios
incluidos en el ZIP de origen, sus contraseñas y vencimientos.
El panel sigue creando/editando/eliminando usuarios mediante commits de GitHub.
El administrador y su vencimiento siguen siendo los de FIXED_ADMIN en
functions/api/updateusers.js (2026-12-16). No se añaden variables de administrador.

La página principal, su diseño y script.js se conservan byte por byte respecto
a la versión API usada como base. Sigue mostrando data.body en el modal o
redirigiendo a data.link. Se conservan todos los asuntos originales, la prioridad
de los códigos sobre los enlaces y la ventana original de 10 minutos.
La forma de obtener el correo pasa de Gmail API/OAuth a IMAP TCP con TLS directo
hacia imap.gmail.com:993. No usa un proxy ni un servicio intermedio.

## 1. Antes de actualizar una instalación existente

Descarga una copia de tu repositorio actual.
Si ya añadiste o cambiaste usuarios desde que enviaste el ZIP de origen, NO
sobrescribas tu script.js actual con el del ZIP: conserva el actual. Este cambio
no necesita modificarlo. Conserva también tus imágenes y estilos actuales.

Sube estos archivos nuevos/modificados respetando sus carpetas:
- admin.html
- mailbox-admin.js
- functions/api/getLastEmail.js
- functions/api/mailboxes.js
- functions/api/updateusers.js
- lib/admin-auth.js
- lib/imap-protocol.js
- lib/imap-mail.js
- lib/mail-rules.js
- lib/mailbox-handler.js
- lib/mailbox-store.js
- lib/mailbox-vault.js
- lib/consult-handler.js
- build.mjs
- package.json
- package-lock.json
- mailboxes.json (solo la primera instalación de esta versión)

Conserva los otros archivos de lib que ya usa tu sistema (users.js, encoding.js).
Si cambiaste la contraseña o vencimiento del administrador después de enviar el
ZIP, conserva esos valores en FIXED_ADMIN al actualizar updateusers.js. Esa es
la única definición usada tanto por usuarios como por buzones.

IMPORTANTE: después de añadir buzones, nunca reemplaces mailboxes.json con el
archivo vacío del ZIP. El archivo del repositorio ya tendrá tu configuración.
No subas node_modules, dist, .env, .dev.vars ni secretos.
Esta versión SÍ incluye un package-lock.json nuevo válido: consérvalo.

## 2. Configuración Cloudflare Pages

Conserva tu proyecto Pages conectado al repositorio GitHub:
- Rama de producción: main
- Framework: None
- Build command: npm run build
- Build output directory: dist
- Directorio raíz: vacío, raíz del repositorio

No necesitas D1 ni KV ni flags Node.js para este código. Las conexiones TCP usan
la API nativa cloudflare:sockets. No elijas el flujo de crear un Worker distinto
ni arrastres el ZIP al panel. El repositorio y las variables son los del mismo
proyecto Pages que ya utilizabas.

## 3. Variables: ahora son cinco necesarias

En Pages > Settings > Variables and Secrets, entorno Production:

| Nombre | Valor | Tipo |
|---|---|---|
| GITHUB_TOKEN | Token real con Contents: Read and write sobre tu repositorio | Secreto |
| GITHUB_REPO | virtualformacion/store-molano, o el propietario/repositorio real conectado a Pages | Texto |
| GITHUB_FILE_PATH | script.js | Texto |
| GITHUB_BRANCH | main | Texto |
| MAILBOX_ENCRYPTION_KEY | Valor aleatorio de al menos 32 caracteres; se recomiendan 64 caracteres hexadecimales | Secreto |

Las cuatro variables GITHUB_* se mantienen como estaban. La clave nueva debe
ser diferente del token de GitHub y de las claves de aplicación. Guárdala también
en tu gestor de contraseñas. Si tienes Node instalado puedes generar un valor
seguro localmente con npm run key; copia ese valor al secreto de Cloudflare.
No ejecutes ese comando como build command y no guardes su salida en GitHub.

GMAIL_CLIENT_ID, GMAIL_CLIENT_SECRET y GMAIL_REFRESH_TOKEN ya NO se utilizan.
Puedes dejarlas temporalmente mientras pruebas; no influyen en esta versión.
El correo y la clave de cada buzón se añaden en el panel, no como variables.
Opcional: GITHUB_MAILBOX_PATH permite otro nombre distinto de mailboxes.json;
no hace falta configurarla y nunca debe apuntar al mismo archivo que script.js.

Guarda las variables y vuelve a desplegar. No cambies MAILBOX_ENCRYPTION_KEY
mientras conserves buzones: de lo contrario sus claves no se podrán descifrar.
Si pierdes esa clave, tendrás que borrar/recrear o volver a introducir las claves
de aplicación de los buzones. Rotar GITHUB_TOKEN no cambia el cifrado.

## 4. Crear una clave de aplicación de Gmail

En la cuenta Google del buzón, activa la verificación en dos pasos y abre:
https://myaccount.google.com/apppasswords

Crea una clave para esta aplicación. Es la clave de 16 letras que muestra Google,
no la contraseña normal del correo. Puedes pegarla con espacios; el panel los
quita al guardarla. Algunas cuentas de organización o Protección Avanzada no
permiten crear claves de aplicación; esa restricción depende de Google.
Si cambias la contraseña principal, Google puede revocar la clave de aplicación.

## 5. Gestión de buzones

Abre /admin.html y entra con tu administrador habitual.
En la sección nueva Buzones Gmail por IMAP:
1. Introduce nombre, correo y clave de aplicación.
2. Deja Carpeta vacía para detectar la carpeta especial Todos los correos de Gmail.
3. Deja Buzón activo marcado y pulsa Guardar buzón.
4. La conexión se comprueba antes de guardar un buzón activo.
5. Pulsa Probar conexión para confirmar el acceso después de cualquier cambio.

Puedes añadir varios buzones, editar su nombre/correo/carpeta/clave, activar,
desactivar, probar y eliminar registros. Una clave vacía al editar conserva la
anterior. Si cambias el correo, debes introducir su nueva clave de aplicación.
Las claves nunca se devuelven al navegador ni se muestran al listar.
Eliminar un registro no borra mensajes ni revoca la clave en Google.

Esta versión admite hasta 10 buzones activos y consulta hasta dos en paralelo,
para acotar conexiones y tiempo de espera de Cloudflare. Puedes conservar otros
buzones desactivados. No reutiliza conexiones entre visitantes.
Los cambios de buzones se leen de GitHub en las nuevas consultas, sin esperar
que acabe el build. Cada commit puede activar un build, igual que los cambios de
usuarios, y queda sujeto a las cuotas de GitHub y Cloudflare.
Si dos administradores guardan a la vez, el SHA de GitHub protege contra
sobrescrituras: recarga la lista y repite si aparece un conflicto.

## 6. Cómo se buscan y muestran los resultados

En cada buzón activo se revisan los últimos 50 mensajes de la carpeta seleccionada,
con la misma ventana original de 10 minutos usando Date (o INTERNALDATE si falta).
Se abre la carpeta en modo EXAMINE y se lee con BODY.PEEK: no se borran correos
ni se marcan como leídos.

Por defecto se detecta Todos los correos por el atributo IMAP \\All, independientemente
del idioma de Gmail. Esto incluye archivados y enviados como la búsqueda normal
de Gmail; Spam y Papelera no se agregan automáticamente. Si Todos los correos
está oculto para IMAP, habilita Mostrar en IMAP en Gmail. También puedes indicar
INBOX en el panel para consultar solamente la bandeja de entrada.

Los resultados de los buzones activos se reúnen y ordenan por fecha. Conservan
la prioridad de código frente a enlace: primero se devuelve el código más reciente
que corresponda; si no hay, el enlace admitido más reciente. Se mantienen los
prefijos de enlaces Netflix y todos los asuntos de STORE MOLANO, sin añadir servicios.
El cuerpo HTML se muestra en el mismo modal de antes y los enlaces conservan la
misma redirección del navegador. El parser MIME lee HTML, texto, Base64 y
quoted-printable. No replica la generación del campo snippet propio de Gmail API:
cuando no hay HTML, utiliza el texto completo disponible en el mensaje.

Destinatarios: se comprueban To, Cc y cabeceras de entrega/reenvío, además de
bloques de destinatario Para:/To: en reenvíos y mensajes adjuntos message/rfc822.
Se reconocen gmail.com/googlemail.com, puntos del nombre Gmail y extensiones +tag.
Las etiquetas + se conservan: usuario+uno y usuario+dos no se confunden.

Un reenvío solo se puede atribuir al destinatario si conserva ese dato en sus
cabeceras, bloque reenviado o mensaje adjunto. Si se eliminó o modificó esa
información, ningún cambio de protocolo permite recuperarla con certeza.
No se consideran suficientes direcciones que aparezcan al azar en el cuerpo.
La cobertura de formatos reenviados depende del mensaje recibido; prueba tus
casos reales antes de dar por completada la migración.

Si un buzón falla, se siguen consultando los demás. Si hay resultado se muestra;
si no hay y algún buzón falló, se avisa que la consulta quedó incompleta.
Cada conexión tiene límite de espera y sesión; se cierran sockets al terminar.
Para controlar memoria se admiten mensajes recientes de hasta 4 MiB y hasta
12 MiB de mensajes por buzón y 24 MiB en total por consulta; un exceso se informa como fallo de
ese buzón. Las cuotas de CPU y ejecución del plan Cloudflare siguen aplicándose.

## 7. Dónde se guardan los datos

| Dato | Ubicación |
|---|---|
| Usuarios y contraseñas | script.js en GitHub, igual que antes |
| Administrador y vencimiento | FIXED_ADMIN en functions/api/updateusers.js, igual que antes |
| Nombre/correo/carpeta/estado de buzones | mailboxes.json en GitHub |
| Claves de aplicación | mailboxes.json, cifradas con AES-256-GCM y un IV aleatorio por guardado |
| Clave que permite descifrar | Secreto MAILBOX_ENCRYPTION_KEY de Cloudflare; nunca en el repositorio |

mailboxes.json no se copia a dist. Aun si el repositorio es público, contendrá
las claves de aplicación cifradas; nombres y correos sí serán visibles en el
repositorio. La contraseña de los usuarios y el administrador siguen con el
modelo original solicitado: no se cambió su autenticación ni su almacenamiento.
El acceso a la configuración de buzones exige las credenciales del administrador.
No se añadió una nueva autenticación del endpoint de consulta de correos.

## 8. Archivos y validación

| Archivo | Cambio |
|---|---|
| functions/api/getLastEmail.js | Punto de entrada IMAP en lugar de Gmail API |
| functions/api/mailboxes.js | Nueva API de gestión y prueba de buzones |
| functions/api/updateusers.js | Exporta el administrador existente para reutilizarlo; CRUD sin cambios |
| admin.html y mailbox-admin.js | Sección nueva de gestión de buzones |
| lib/admin-auth.js | Reutiliza el administrador y su vencimiento |
| lib/imap-protocol.js | Cliente IMAP sobre TCP TLS 993, lectura de literales, cierre y tiempos de espera |
| lib/imap-mail.js y lib/mail-rules.js | Lectura MIME, destinatarios y selección de resultados con asuntos originales |
| lib/mailbox-store.js | Lectura/escritura del archivo de buzones mediante commits GitHub con SHA |
| lib/mailbox-vault.js | Validación y cifrado/descifrado de claves de aplicación |
| lib/mailbox-handler.js y lib/consult-handler.js | Operaciones del panel y consulta de varios buzones |
| mailboxes.json | Configuración inicial vacía, solo instalar una vez |
| build.mjs, package.json y package-lock.json | Incluyen el JS del panel y la dependencia postal-mime |
| script.js, index.html, styles.css e imágenes | Sin cambios respecto a la base API |

npm test comprueba CRUD original de usuarios, cifrado autenticado, almacenamiento
GitHub, claves ocultas, conflictos, cambios de estado, reenvíos de texto y HTML,
message/rfc822, alias, todos los asuntos/enlaces originales, prioridad de resultados
y el protocolo IMAP con un socket simulado que entrega datos fragmentados.
La compilación con Wrangler comprueba los imports y el empaquetado de Pages.

No se utilizaron tus credenciales reales, no se crearon commits en tu repositorio
y no se probó la conexión con tu Gmail desde un despliegue real. La validación cubre pruebas automatizadas con respuestas simuladas y la
compilación para Cloudflare, no una conexión a tu buzón real. Después de subir:
1. Prueba una conexión desde el panel.
2. Consulta un código directo, un reenvío y una dirección +tag que utilices.
3. Prueba una redirección Netflix.
4. Comprueba editar/desactivar/eliminar un buzón y cambiar un usuario.
5. Mantén una copia de la versión API para volver atrás si un caso real difiere.

Fuentes técnicas oficiales:
https://developers.cloudflare.com/workers/runtime-apis/tcp-sockets/
https://support.google.com/accounts/answer/185833
https://developers.google.com/workspace/gmail/imap/imap-extensions
https://github.com/postalsys/postal-mime
