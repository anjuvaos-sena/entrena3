# ENI Training

## Almacenamiento de PDFs de inscripciones en Supabase

En producción, los archivos adjuntos del formulario de inscripciones se guardan en Supabase Storage mediante su API compatible con S3. La integración usa el bucket privado `eni-bucket`; en desarrollo local, Django continúa guardando los archivos en `media/`.

La configuración está en `pr_eni/settings.py`. El campo de archivo de `app_inscripciones` ya usa el almacenamiento configurado por Django, por lo que no se necesita cambiar el formulario ni migrar la base de datos para activar el nuevo destino.

### 1. Verificar el proyecto y el bucket en Supabase

En la cuenta `anjuvaos-sena`, abre el proyecto `entrena3sb` y confirma en **Storage** que exista el bucket `eni-bucket`. Si no existe, créalo con ese nombre y déjalo **privado**. No habilites el acceso público para almacenar documentos de inscripción.

> La existencia del proyecto y del bucket no pudo comprobarse desde esta sesión: no estuvo disponible un MCP de Supabase. Confírmala en el dashboard antes de desplegar.

### 2. Crear credenciales para S3

Desde la configuración de Storage del proyecto, crea un par de claves S3 (Access Key ID y Secret Access Key). Usa credenciales dedicadas a esta aplicación y no las claves `anon` ni `service_role` de la API de Supabase. Trata la clave secreta como una contraseña: no la guardes en el repositorio, en este archivo ni en logs.

El endpoint de S3 tiene este formato:

```text
https://<PROJECT_REF>.storage.supabase.co/storage/v1/s3
```

Reemplaza `<PROJECT_REF>` por la referencia del proyecto `entrena3sb` que aparece en su dashboard. Para esta conexión, la región S3 es `us-east-1`.

### 3. Configurar variables de entorno en Render

En **Render → servicio web → Environment**, agrega las siguientes variables. Configura las credenciales en los campos protegidos de Render, no en el código:

| Variable | Valor |
| --- | --- |
| `DEBUG` | `False` |
| `SUPABASE_S3_ENDPOINT` | Endpoint del proyecto, por ejemplo `https://<PROJECT_REF>.storage.supabase.co/storage/v1/s3` |
| `SUPABASE_S3_ACCESS_KEY_ID` | Access Key ID S3 generado en Supabase |
| `SUPABASE_S3_SECRET_ACCESS_KEY` | Secret Access Key S3 generado en Supabase |
| `SUPABASE_S3_BUCKET` | `eni-bucket` |
| `SUPABASE_S3_REGION` | `us-east-1` |

Configura también `SECRET_KEY`, `DATABASE_URL` y las demás variables de producción que ya requiera el servicio. No compartas ni reutilices las claves S3 fuera del servicio.

Con `DEBUG=False`, Django selecciona el almacenamiento S3 para archivos cargados y conserva WhiteNoise para archivos estáticos. Si faltan el endpoint o alguna de las dos claves, el proceso falla al iniciar con un error que identifica las variables requeridas; no usará silenciosamente el disco efímero de Render. En desarrollo (`DEBUG=True`) no se requieren claves S3.

### 4. Desplegar

Configura el servicio web en Render con `bash build.sh` como **Build Command** y `gunicorn pr_eni.wsgi:application` como **Start Command** (conserva el comando de inicio existente si ya es equivalente). El script de build instala `requirements.txt`, recopila estáticos y aplica migraciones. Después de agregar la dependencia de almacenamiento y las variables de entorno:

1. Guarda las variables en Render y ejecuta un nuevo deploy.
2. Revisa los logs del build y del arranque. La carga de claves fallida o un endpoint incorrecto aparecerán como errores de almacenamiento.
3. Verifica que el bucket permanezca privado y que las credenciales generadas tengan acceso S3 al bucket correcto.

No se requiere ejecutar una migración nueva: cambiar el backend de almacenamiento no modifica el esquema del modelo `Inscripcion`.

### 5. Probar la carga

1. Inicia sesión con un usuario autorizado del grupo **Instructores** y abre una convocatoria publicada.
2. Envía una inscripción con un PDF válido.
3. Comprueba la confirmación en la aplicación y, desde el dashboard de Supabase, que se haya creado un objeto bajo `requisitos_inscripciones/` en `eni-bucket`.
4. Comprueba que el objeto no sea accesible anónimamente. Django genera URLs firmadas y temporales cuando se solicita la URL del archivo; no compartas esas URLs fuera de los flujos autorizados.

La aplicación guarda la referencia del objeto en la base de datos. Los PDFs existentes en el almacenamiento local no se copian automáticamente al bucket; si hay archivos que deban conservarse, migra esos objetos por separado y verifica sus referencias antes de retirar el almacenamiento anterior.
