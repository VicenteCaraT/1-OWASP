# Trabajo práctico: Seguridad OWASP con NodeGoat

> **Plataforma elegida para continuar:** NodeGoat, porque es la aplicación usada en clase. El historial de WebGoat se conserva al final de este documento.

## Datos de la práctica

- **Aplicación:** OWASP NodeGoat, ejecutada localmente con Docker Compose.
- **URL:** <http://127.0.0.1:4000>
- **Tutorial integrado:** <http://127.0.0.1:4000/tutorial>
- **Estado observado:** la pantalla de inicio de sesión carga correctamente; evidencia compartida en la conversación (guardar como `evidencias/nodegoat-inicio.png`).
- **Fecha:** completar al realizar la práctica.
- **Estudiante:** completar.
- **Objetivo:** mostrar la aplicación y documentar cinco ataques ejecutados en el entorno de laboratorio.

> Ejecutar los ataques solo contra la copia local de NodeGoat. Usar únicamente usuarios y datos ficticios provistos por la aplicación. Evitar pruebas de denegación de servicio, lectura de archivos, acceso a servicios externos y cambios destructivos.

## Puesta en marcha

El repositorio oficial incluye un `Dockerfile` y `docker-compose.yml` que levantan NodeGoat y MongoDB.

1. En una terminal, ir a una carpeta de trabajo y clonar el repositorio:

   ```bash
   git clone https://github.com/OWASP/NodeGoat.git
   cd NodeGoat
   ```

2. Antes de iniciar el contenedor, abrir `docker-compose.yml` y cambiar el mapeo del servicio `web` de:

   ```yaml
   - "4000:4000"
   ```

   a:

   ```yaml
   - "127.0.0.1:4000:4000"
   ```

   Así la aplicación queda disponible solo desde la computadora local.

3. Construir e iniciar los servicios:

   ```bash
   docker compose build
   docker compose up
   ```

4. Abrir <http://127.0.0.1:4000>. El tutorial está en <http://127.0.0.1:4000/tutorial>.
5. Para detener NodeGoat, presionar `Ctrl+C` en la terminal donde corre Compose.

La guía del repositorio indica que los usuarios de prueba sembrados son `user1` / `User1_123`, `user2` / `User2_123` y `admin` / `Admin_123`. Usar esas cuentas únicamente en la copia local del laboratorio.

## Seguimiento

- [x] Ataque 1: Inyección NoSQL en filtro de asignaciones.
- [x] Ataque 2: XSS almacenado en perfil.
- [x] Ataque 3: IDOR al consultar asignaciones de otro usuario de prueba.
- [x] Ataque 4: Falta de control de acceso por función en `/benefits`.
- [x] Ataque 5: CSRF en formulario de perfil.

Marcar cada ataque como ejecutado solo después de observar el resultado y guardar las capturas. El tutorial de NodeGoat explica estas categorías en las secciones A1, A3, A4, A7 y A8.

---

## Ataque 1 — Inyección NoSQL

**¿En qué consiste?:** la aplicación incorpora un valor ingresado por el usuario en una consulta MongoDB sin validarlo correctamente.

**¿Qué hará la prueba?:** enviar una entrada preparada en el filtro de asignaciones para cambiar la condición de búsqueda y hacer que devuelva registros de más de una cuenta de laboratorio.

**Impacto posible:** exposición de información de otros usuarios y pérdida de confidencialidad.

**Objetivo:** alterar el filtro de asignaciones para que devuelva datos de más de un usuario.

**Ruta de la aplicación:** iniciar sesión como `user1` → abrir `My Stocks` / `Allocations` → localizar el campo `Stocks Threshold`.

### Pasos

1. Abrir el tutorial en `/tutorial` y revisar la sección **A1 - Injection**, que incluye **SQL and NoSQL Injection**.
2. En la aplicación, iniciar sesión con el usuario de prueba `user1` y contraseña `User1_123`.
3. Abrir la página de asignaciones (`/allocations`) y anotar qué filas aparecen con un umbral normal.
4. En `Stocks Threshold`, probar el valor que documenta el tutorial de NodeGoat:

   ```text
   1'; return 1 == '1
   ```

5. Enviar el filtro y observar si ahora aparecen asignaciones de más de un usuario.
6. Guardar capturas del filtro normal, el valor de prueba y el resultado. No usar la variante de bucle infinito mencionada en materiales antiguos.

**Línea base observada en la captura:** la página `Allocations` muestra los activos de John Doe: Domestic Stocks 37 %, Funds 25 %, Bonds 38 %. El campo `Stocks Threshold` aparece vacío; no se observa un payload enviado.

**Resultado observado:** exitoso. Después de enviar `1'; return 1 == '1`, la página mostró asignaciones de **Node Goat Admin**, **John Doe** y **Will Smith**. El filtro permitió consultar asignaciones de otros usuarios.

| Momento | Usuario mostrado | Domestic Stocks | Funds | Bonds |
|---|---|---:|---:|---:|
| Antes del ataque | John Doe | 37 % | 25 % | 38 % |
| Después del ataque | Node Goat Admin | 25 % | 30 % | 45 % |
| Después del ataque | John Doe | 37 % | 25 % | 38 % |
| Después del ataque | Will Smith | 19 % | 17 % | 64 % |

**Evidencia:** línea base en `evidencias/ataque-1-nosql-linea-base.png`; resultado en `evidencias/ataque-1-nosql-resultado.png`.

### Bitácora de ejecución

1. Se inició sesión y se abrió **Allocations**.
2. Antes de enviar una entrada, se registró la línea base: solo aparecía John Doe con Domestic Stocks 37 %, Funds 25 % y Bonds 38 %.
3. Se escribió `1'; return 1 == '1` en **Stocks Threshold** y se presionó **Submit**.
4. La respuesta mostró también las asignaciones de Node Goat Admin y Will Smith, además de John Doe. Se confirmó exposición de registros de otras cuentas de prueba.
5. Se guardaron las capturas de línea base y resultado indicadas arriba.

**Impacto:** una entrada altera la expresión de consulta y puede exponer registros ajenos.

**Mitigación:** no construir consultas con texto sin validar; validar el tipo y formato del umbral y evitar operadores de evaluación del lado de MongoDB.

---

## Ataque 2 — XSS almacenado en el perfil

**¿En qué consiste?:** contenido ingresado por una persona se guarda y luego se interpreta como código activo en el navegador de quien ve la página.

**¿Qué hará la prueba?:** guardar una alerta JavaScript inocua como nombre del perfil y comprobar que se ejecuta al mostrar o recargar el perfil.

**Impacto posible:** ejecución de código en el navegador de otras personas que visiten el contenido; según el contexto, puede alterar la página o realizar acciones con su sesión.

**Objetivo:** comprobar si un valor persistido en el perfil se interpreta como HTML/JavaScript al volver a mostrarlo.

**Ruta de la aplicación:** perfil / edición de perfil.

### Pasos

1. En el tutorial, revisar **A3 - Cross Site Scripting** y su ejemplo del perfil.
2. Iniciar sesión con `user1` / `User1_123` y abrir la edición de perfil.
3. En un campo de nombre de prueba, ingresar una alerta inocua:

   ```html
   <script>alert('NodeGoat XSS')</script>
   ```

4. En `Bank Routing #`, ingresar el formato ficticio válido que da el mismo formulario: `0198212#` (no confundirlo con el campo `Bank Account #`).
5. Presionar **Submit**. Si el navegador muestra la alerta `NodeGoat XSS`, guardarla como evidencia.
6. Aceptar la alerta y recargar el perfil para verificar si vuelve a ejecutarse; eso confirmaría que el valor quedó almacenado.
7. Después de guardar la evidencia, restaurar `First Name` a `John` y dejar el perfil con datos ficticios válidos.

**Intentos observados:**

1. Se ingresó `<script>alert('NodeGoat XSS')</script>` en `First Name` y se envió el formulario. NodeGoat rechazó el envío porque `Bank Routing #` no tenía el formato esperado.
2. Se escribió `0198212#` en `Bank Account #` por error; el formulario volvió a mostrar el error de `Bank Routing #`.
3. Se corrigió la ubicación y se ingresó el formato de ejemplo `0198212#` en `Bank Routing #`. Al enviar, el navegador mostró una alerta con el texto `NodeGoat XSS` desde `127.0.0.1:4000`.

La tercera captura confirmó ejecución de JavaScript. Después, el usuario informó que la alerta vuelve a aparecer cada vez que recarga el perfil y que la aplicación muestra `Profile updated successfully`. Esto confirma que el contenido quedó almacenado y se ejecuta al volver a renderizar el perfil.

**Para continuar:** conservar el payload en `First Name`; ingresar `0198212#` en el campo **Bank Routing #** (no en `Bank Account #`) y volver a presionar **Submit**. Si aparece otro error de validación, anotarlo antes de cambiar otros campos.

**Resultado observado del ataque:** exitoso. El mensaje `Profile updated successfully` confirmó que se guardó el perfil, y la alerta `NodeGoat XSS` reapareció tras cada recarga.

**Evidencia:** validaciones rechazadas en `evidencias/ataque-2-xss-validacion.png` y `evidencias/ataque-2-xss-validacion-2.png`; alerta ejecutada en `evidencias/ataque-2-xss-alerta.png`. Reaparición tras recargar confirmada por el usuario; captura de esa recarga pendiente.

**Limpieza realizada y verificada:** se reemplazó el `First Name` por `John`, se dejó vacío `Bank Account #` y se conservó `0198212#` en `Bank Routing #` para cumplir la validación. La captura muestra `Profile updated successfully` y no muestra una alerta. El usuario recargó el perfil y confirmó que la alerta no reapareció.

**Evidencia de limpieza:** `evidencias/ataque-2-xss-limpieza.png`.

**Impacto:** contenido almacenado se ejecuta en el navegador al mostrarse.

**Mitigación:** codificar la salida en su contexto y sanitizar el contenido cuando se admita HTML.

---

## Ataque 3 — IDOR en las asignaciones

**¿En qué consiste?:** el servidor usa un identificador que viene en la dirección de la página, pero no comprueba correctamente que el usuario conectado tenga permiso para ver ese recurso.

**¿Qué hará la prueba?:** iniciar sesión como `user1` y cambiar el ID de sus asignaciones por el ID de `user2`, sin editar ningún dato.

**Impacto posible:** acceso no autorizado a información de otra cuenta, afectando la confidencialidad.

**Objetivo:** verificar si un usuario autenticado puede ver asignaciones de otro usuario cambiando el identificador de la URL.

**Ruta documentada:** `/allocations/{id}`.

### Pasos

1. En el tutorial, revisar **A4 - Insecure Direct Object Reference**.
2. Iniciar sesión como `user2` / `User2_123`, abrir sus asignaciones y copiar únicamente el identificador numérico de la ruta `/allocations/{id}`. No modificar registros.
3. Cerrar sesión e iniciar como `user1` / `User1_123`.
4. Abrir las asignaciones propias y anotar el ID y los datos visibles.
5. En la barra de direcciones, sustituir el ID propio por el ID de prueba de `user2` que anotaste en el paso 2.
6. Comprobar si la página muestra los datos de `user2` o deniega el acceso. Guardar capturas de ambas situaciones.

**ID de John Doe (`user1`):** `2` (confirmado en la captura previa).

**ID probado:** `3`, asociado a Will Smith (confirmado en la captura).

**Resultado observado:** exitoso. La captura inicial de `/allocations/2` muestra las asignaciones de John Doe (Domestic Stocks 37 %, Funds 25 %, Bonds 38 %). Al cambiar el identificador a `/allocations/3`, la aplicación muestra las de Will Smith (Domestic Stocks 19 %, Funds 17 %, Bonds 64 %), sin cambiar de sesión. Esto confirma acceso a los datos de otra cuenta mediante la manipulación del ID.

**Evidencia:** antes en `evidencias/ataque-3-idor-antes.png`; después en `evidencias/ataque-3-idor-despues.png`.

**Impacto:** la aplicación confía en un identificador controlado por el cliente sin verificar que el recurso pertenece al usuario autenticado.

**Mitigación:** autorizar cada solicitud en el servidor usando la identidad de sesión y el recurso solicitado.

---

## Ataque 4 — Falta de control de acceso por función

**¿En qué consiste?:** una función administrativa no verifica en el servidor que quien la solicita tenga rol de administrador.

**¿Qué hará la prueba?:** entrar con una cuenta normal e intentar abrir la ruta `/benefits` directamente, aunque el menú no muestre esa opción.

**Impacto posible:** usuarios sin permisos podrían ver o modificar información o funciones administrativas.

**Objetivo:** comprobar si una cuenta normal puede abrir una función administrativa cuya opción no aparece en el menú.

**Ruta documentada:** `/benefits`.

### Pasos

1. En el tutorial, revisar **A7 - Missing Function Level Access Control**.
2. Cerrar sesión e iniciar como usuario normal: `user1` / `User1_123`.
3. Sin usar una cuenta de administrador, escribir directamente en el navegador:

   ```text
   http://127.0.0.1:4000/benefits
   ```

4. Observar si se muestra el módulo de beneficios o si el servidor redirige/deniega el acceso.
5. Para la demostración, limitarse a visualizar el módulo. No guardar cambios de beneficios salvo que la consigna docente lo exija y se trate de datos ficticios.
6. Guardar una captura donde se vea que la sesión es de `user1` y el resultado al abrir `/benefits`.

**Resultado observado:** exitoso. Desde la sesión de usuario normal, al abrir directamente `/benefits`, el servidor mostró la tabla de beneficios con John Doe (ID 2) y Will Smith (ID 3), incluidas sus fechas de inicio. No se presionaron los botones `Save` ni se modificaron datos.

**Evidencia:** `evidencias/ataque-4-control-acceso.png`.

**Impacto:** ocultar un enlace en la interfaz no protege una función si el servidor no valida el rol.

**Mitigación:** verificar autorización en cada endpoint y operación del lado del servidor.

---

## Ataque 5 — CSRF en el perfil

**¿En qué consiste?:** un sitio externo induce al navegador de una persona autenticada a enviar una solicitud que cambia datos en la aplicación, aprovechando las credenciales de sesión que el navegador adjunta.

**¿Qué hará la prueba?:** desde una página local de demostración, enviar un formulario a NodeGoat mientras la sesión de prueba está abierta y observar si modifica los datos ficticios del perfil.

**Impacto posible:** cambios no autorizados en información de la cuenta mientras la víctima mantiene una sesión activa.

**Objetivo:** demostrar, únicamente en el laboratorio local, que un formulario externo puede intentar enviar una modificación de perfil usando la sesión abierta.

**Ruta documentada:** `POST /profile`.

### Preparación del formulario de prueba

1. En el tutorial, revisar **A8 - Cross Site Request Forgery**. Allí se explica el formulario falso que apunta a `/profile`.
2. Abrir el archivo local `csrf-demo.html` preparado junto a este seguimiento. Sirve una forma visible de demostración y apunta únicamente a NodeGoat local. El `bankRouting` usa el formato válido que exige el perfil (`0198212#`).

   ```html
   <!doctype html>
   <html lang="es">
   <meta charset="utf-8">
   <title>Prueba CSRF local</title>
   <body>
     <p>Formulario de demostración para NodeGoat local.</p>
     <form method="POST" action="http://127.0.0.1:4000/profile">
       <input type="hidden" name="bankAcc" value="9999999">
       <input type="hidden" name="bankRouting" value="0198212#">
       <button type="submit">Enviar cambios de prueba a NodeGoat local</button>
     </form>
   </body>
   </html>
   ```

3. Iniciar sesión en NodeGoat como `user1` y abrir su perfil. Anotar los valores de prueba actuales para poder restaurarlos; ahora `Bank Account #` está vacío y `Bank Routing #` tiene `0198212#`.
4. Servir el archivo desde localhost en una terminal, dentro de la carpeta donde está guardado:

   ```bash
   python3 -m http.server 5000 --bind 127.0.0.1
   ```

5. Abrir `http://127.0.0.1:5000/csrf-demo.html` en otra pestaña del mismo navegador y presionar el botón una sola vez.
6. Volver al perfil de NodeGoat y observar si los campos bancarios ficticios cambiaron. Restaurar los valores originales al terminar.
7. Si el navegador o la aplicación bloquea la solicitud, registrar el mensaje. No desactivar controles de seguridad del navegador para forzar el resultado.
8. Guardar capturas del formulario local y del resultado en el perfil.

**Resultado observado:** exitoso. Desde la página local `127.0.0.1:5000/csrf-demo.html` se envió el formulario al perfil de NodeGoat en `127.0.0.1:4000`. NodeGoat respondió `Profile updated successfully` y el campo `Bank Account #` quedó en `9999999`. La captura posterior muestra la sesión de Will Smith; por lo tanto, esa parece ser la cuenta cuyo perfil recibió la modificación. No se envió un token CSRF en el formulario de prueba.

**Evidencia:** página de prueba en `evidencias/ataque-5-csrf-forma.png`; respuesta del servidor en `evidencias/ataque-5-csrf-respuesta.png`; perfil resultante en `evidencias/ataque-5-csrf-resultado.png`.

**Limpieza realizada:** el usuario confirmó que limpió los campos bancarios de prueba del perfil de Will Smith (`_id: 3`). El seed oficial lo crea con nombre `Will Smith` y sin `bankAcc` ni `bankRouting`. La limpieza se registró con este comando, que elimina solo esos dos campos sin reiniciar la base entera:

```bash
docker compose exec mongo mongo nodegoat --eval 'db.users.updateOne({_id: 3}, {$unset: {bankAcc: "", bankRouting: ""}})'
```

Después de la limpieza, recargar `/profile` y comprobar que el encabezado diga Will Smith y que `Bank Account #` y `Bank Routing #` estén vacíos. Fuente de los datos de inicio: [artifacts/db-reset.js](https://github.com/OWASP/NodeGoat/blob/master/artifacts/db-reset.js). **Estado del servidor de demostración en el puerto 5000:** confirmar que se detuvo con `Ctrl+C`.

**Impacto:** una solicitud iniciada desde otra página puede modificar datos aprovechando una sesión autenticada, si las protecciones CSRF y cookies lo permiten.

**Mitigación:** tokens CSRF por solicitud, cookies `SameSite` y validación de origen como capas adicionales.

---

## Conclusión y evidencias

- **Ataques que se observaron con éxito:** pendiente.
- **Ataques bloqueados o que no se reprodujeron:** pendiente.
- **Conclusión personal:** pendiente.

### Capturas

- [ ] Ataque 1 — inyección NoSQL: `evidencias/ataque-1-nosql.png`
- [ ] Ataque 2 — XSS almacenado: `evidencias/ataque-2-xss.png`
- [ ] Ataque 3 — IDOR: `evidencias/ataque-3-idor.png`
- [ ] Ataque 4 — control de acceso: `evidencias/ataque-4-control-acceso.png`
- [ ] Ataque 5 — CSRF: `evidencias/ataque-5-csrf.png`

En cada captura, procurar que se vea NodeGoat y la evidencia del resultado, sin incluir datos personales reales.

## Fuentes oficiales

- [Repositorio y README de OWASP NodeGoat](https://github.com/OWASP/NodeGoat)
- [A1 — Injection](https://github.com/OWASP/NodeGoat/blob/master/app/views/tutorial/a1.html)
- [A3 — Cross-Site Scripting](https://github.com/OWASP/NodeGoat/blob/master/app/views/tutorial/a3.html)
- [A4 — Insecure Direct Object Reference](https://github.com/OWASP/NodeGoat/blob/master/app/views/tutorial/a4.html)
- [A7 — Missing Function Level Access Control](https://github.com/OWASP/NodeGoat/blob/master/app/views/tutorial/a7.html)
- [A8 — Cross-Site Request Forgery](https://github.com/OWASP/NodeGoat/blob/master/app/views/tutorial/a8.html)

---

# Historial de la práctica con WebGoat

Esta sección conserva los pasos y observaciones de la sesión con WebGoat. La práctica continúa con NodeGoat según lo acordado.

## Puesta en marcha y acceso

- Se inició el contenedor `webgoat/webgoat` con los puertos publicados en `127.0.0.1:8080` y `127.0.0.1:9090`.
- `docker ps` mostró el contenedor activo con estado `unhealthy`. Sin embargo, los registros mostraron `Tomcat started on port 8080`, `Started WebGoat` y el mensaje para abrir `http://127.0.0.1:8080/WebGoat`.
- La primera URL con barra final (`/WebGoat/`) mostró un error 404. Se indicó probar sin barra final: <http://127.0.0.1:8080/WebGoat>. El usuario confirmó que pudo ingresar.

## Recorrido observado: `SQL Injection (intro)`

Ruta en el menú: `(A3) Inyección` → `SQL Injection (intro)`. En esta versión las actividades están dentro de la lección y se recorren con los círculos numerados.

### Actividad de consulta SELECT

- **Consigna observada:** recuperar el departamento de Bob Franco desde la tabla de ejemplo.
- **Consulta indicada:**

  ```sql
  SELECT department FROM employees WHERE first_name = 'Bob' AND last_name = 'Franco';
  ```

- **Resultado esperado según la tabla:** `Marketing`.
- **Estado:** la siguiente captura mostró el indicador del paso 2 en verde; el valor de salida no quedó registrado en el chat.
- **Evidencia:** pendiente de guardar como `evidencias/webgoat-sqli-select.png`.

### Actividad DML

- **Consigna observada:** cambiar el departamento de Tobi Barnett a `Sales`.
- **Consulta indicada:**

  ```sql
  UPDATE employees SET department = 'Sales' WHERE first_name = 'Tobi' AND last_name = 'Barnett';
  ```

- **Estado:** la captura posterior mostró el avance al paso DDL y los indicadores anteriores en verde; se toma como completado por el progreso de la lección. No se compartió una captura del mensaje de resultado.
- **Evidencia:** pendiente de guardar como `evidencias/webgoat-sqli-dml.png`.
- **Nota de alcance:** esta actividad permite ejecutar DML sobre datos ficticios con privilegios amplios. Es una demostración de impacto en el laboratorio; no demuestra por sí sola una inyección desde un formulario de usuario corriente.

### Actividad DDL pendiente

- **Consigna observada:** agregar una columna `phone` de tipo `varchar(20)` a la tabla `employees`.
- **Consulta indicada, no confirmada como ejecutada:**

  ```sql
  ALTER TABLE employees ADD COLUMN phone VARCHAR(20);
  ```

- **Estado:** pendiente; no llegó una captura del resultado antes de cambiar la plataforma.
- **Evidencia:** `evidencias/webgoat-sqli-ddl.png` (pendiente).

## Ataques inicialmente propuestos para WebGoat

Estos fueron los cinco ataques considerados antes de cambiar a NodeGoat; no se reportaron como ejecutados:

- [ ] SQL injection en una lección de inyección interactiva.
- [ ] XSS reflejado.
- [ ] XSS almacenado.
- [ ] IDOR / control de acceso.
- [ ] XXE.

Se había sugerido un payload clásico para una antigua etapa de login SQLi, pero se corrigió antes de usarlo: **no se confirmó ni se registró su ejecución** porque la versión abierta tenía otra actividad.

## Fuentes de WebGoat

- [Repositorio y README oficial de OWASP WebGoat](https://github.com/WebGoat/WebGoat)
- [Lección de SQL Injection](https://github.com/WebGoat/WebGoat/tree/main/src/main/java/org/owasp/webgoat/lessons/sqlinjection)
