# Guia de Pruebas de Seguridad (Paso a Paso)

Basado en los hallazgos de `hallazgos_seguridad.md`.

## Objetivo

Validar de forma controlada cada vulnerabilidad encontrada para:

- Confirmar el fallo (reproducibilidad)
- Levantar bug con evidencia tecnica
- Verificar luego la correccion (regresion)

---

## Alcance y advertencia

- Esta guia es **solo para tu sistema** y ambiente autorizado.
- Ejecuta pruebas en entorno local o staging, no en produccion.
- No uses estas tecnicas fuera de sistemas propios/autorizados.

---

## Preparacion del entorno

## 1) Datos base

- URL base local: `http://127.0.0.1:8000`
- Usuario admin (ejemplo actual): `admin@cafetin.com / password`
- Crea al menos 2 clientes de prueba:
  - `cliente1@test.com`
  - `cliente2@test.com`
- Crea pedidos con ambos clientes para probar IDOR.

## 2) Herramientas

- Insomnia (para requests HTTP)
- Navegador (para inspeccionar cookies/CSRF mas facil)
- Opcional: Postman/Newman, Burp Community, OWASP ZAP

## 3) Nota importante sobre Laravel + CSRF (rutas web)

Tus endpoints estan en `web.php`, por lo que operaciones `POST/PUT/DELETE` requieren:

- Cookie de sesion (`laravel_session`)
- Cookie `XSRF-TOKEN`
- Header `X-XSRF-TOKEN` o campo `_token`

Para simplificar pruebas en Insomnia:

1. Haz login primero en navegador.
2. Copia cookies (`laravel_session`, `XSRF-TOKEN`) al Cookie Jar de Insomnia.
3. En requests mutables agrega header:
   - `X-XSRF-TOKEN: <valor_decodificado_de_XSRF-TOKEN>`
4. Mantener `Cookie` automaticamente en Insomnia.

---

## Plantilla de evidencia (usar por cada prueba)

- ID prueba:
- Hallazgo asociado:
- Precondiciones:
- Request (metodo, URL, headers, body):
- Respuesta esperada segura:
- Respuesta real observada:
- Resultado: `Vulnerable` / `No vulnerable`
- Capturas/adjuntos:
- Severidad sugerida:

---

## Prueba 1 - Escalada de privilegios en cambio de estado (Critico)

Hallazgo: cliente puede cambiar estado de cualquier pedido.

## Precondiciones

- Tener sesion iniciada como `cliente`.
- Debe existir un pedido (idealmente de otro usuario).

## Pasos (Insomnia)

1. Crear request:
   - Metodo: `PUT`
   - URL: `{{base_url}}/pedidos/1/estado`
2. Headers:
   - `Content-Type: application/x-www-form-urlencoded`
   - `X-XSRF-TOKEN: <token>`
3. Body (Form URL Encoded):
   - `estado=cancelado` (o `listo` / `entregado`)
4. Enviar request.

## Resultado esperado seguro

- HTTP `403 Forbidden` para rol cliente.

## Resultado vulnerable

- HTTP `200/302` y estado del pedido cambia.

## Verificacion adicional

- Abrir `/pedidos/1` o listado y confirmar que el estado efectivamente cambio.

---

## Prueba 2 - IDOR en listado de pedidos (Alto)

Hallazgo: cliente ve pedidos globales.

## Precondiciones

- Pedidos creados por distintos usuarios.
- Sesion como `cliente1`.

## Pasos (navegador o Insomnia GET)

1. `GET {{base_url}}/pedidos`
2. Revisar si aparecen pedidos de `cliente2` o admin.

## Resultado esperado seguro

- Cliente solo ve sus propios pedidos.

## Resultado vulnerable

- Cliente ve pedidos de terceros (codigo, total, estado, pago, etc.).

---

## Prueba 3 - IDOR en detalle de pedido (Alto)

Hallazgo: cliente accede a pedido ajeno por ID secuencial.

## Precondiciones

- Conocer/estimar ID de pedido de otro usuario (ej. `2`).
- Sesion como cliente no propietario.

## Pasos

1. `GET {{base_url}}/pedidos/2`
2. Repetir con IDs `1..N`.

## Resultado esperado seguro

- `403` o `404` para pedidos ajenos.

## Resultado vulnerable

- Devuelve vista detalle con datos del pedido ajeno.

---

## Prueba 4 - Login de usuario inactivo (Alto)

Hallazgo: usuario con `estado=inactivo` aun autentica.

## Precondiciones

- Marcar un usuario como `inactivo` desde admin.

## Pasos

1. Cerrar sesion actual.
2. Login con usuario inactivo:
   - `POST {{base_url}}/login`
   - email/password del usuario inactivo.
3. Observar redireccion.

## Resultado esperado seguro

- Login rechazado con mensaje de credenciales invalidas o usuario inactivo.

## Resultado vulnerable

- Login exitoso y acceso al dashboard.

---

## Prueba 5 - Falta de rate limiting en endpoints sensibles (Medio)

Hallazgo: solo login tiene throttling.

## Endpoints a probar

- `POST /pedidos`
- `PUT /pedidos/{id}/estado`
- `PUT /pagos/{id}/confirmar`

## Metodo A (Insomnia manual)

1. Duplicar request sensible varias veces.
2. Disparar rapidamente (burst manual).
3. Ver si aparece `429 Too Many Requests`.

## Metodo B (mas fuerte, recomendado)

Usar script (k6, autocannon o bash loop) para 50-200 requests.

Ejemplo conceptual:

- Enviar 100 requests en 10 segundos al endpoint.
- Medir:
  - porcentaje de `2xx`
  - `429` recibidos
  - tiempo de respuesta

## Resultado esperado seguro

- El sistema limita con `429` despues de cierto umbral.

## Resultado vulnerable

- Procesa casi todos los requests sin limite.

---

## Prueba 6 - Hardening HTTP y sesion (Medio)

Hallazgo: faltan headers defensivos y ajustes de cookies/produccion.

## 6.1 Verificar headers de seguridad

### Pasos (Insomnia o curl)

1. `GET {{base_url}}/dashboard` (autenticado)
2. Revisar response headers.

### Headers recomendados a validar

- `Content-Security-Policy`
- `Strict-Transport-Security` (si HTTPS)
- `X-Frame-Options` o `frame-ancestors` (CSP)
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy`

### Resultado vulnerable

- No aparecen uno o varios headers clave.

## 6.2 Verificar cookies de sesion

### Pasos

1. Inspeccionar `Set-Cookie` de `laravel_session`.
2. Confirmar flags:
   - `HttpOnly`
   - `Secure` (en HTTPS)
   - `SameSite`

### Resultado vulnerable

- Cookie sin `Secure` en ambiente HTTPS productivo.

## 6.3 Verificar modo debug

### Pasos

1. Forzar error controlado (ruta inexistente o excepcion conocida).
2. Revisar si muestra stack trace y detalles internos.

### Resultado vulnerable

- Muestra trazas detalladas en entorno que simula produccion.

---

## Matriz de priorizacion de fixes (sugerida)

1. **Inmediato (P0):**
   - Escalada de privilegios en `/pedidos/{id}/estado`
   - IDOR en `pedidos index/show`
2. **Corto plazo (P1):**
   - Bloqueo de login para `estado=inactivo`
   - Rate limiting endpoints sensibles
3. **Mediano plazo (P2):**
   - Hardening headers HTTP + CSP + politicas de cookie en produccion

---

## Checklist de regresion (despues de corregir)

- [ ] Cliente recibe `403` al cambiar estado de pedidos.
- [ ] Cliente solo lista sus pedidos.
- [ ] Cliente no puede abrir detalle de pedido ajeno (`403/404`).
- [ ] Usuario inactivo no puede iniciar sesion.
- [ ] Endpoints sensibles responden `429` bajo rafagas.
- [ ] Headers de seguridad presentes.
- [ ] En produccion: `APP_DEBUG=false`, cookie de sesion segura.

---

## Sugerencia para bug report interno

Usa este formato para Jira/Linear/GitHub Issues:

- **Titulo:** `[SEC][ALTO] IDOR en detalle de pedidos permite acceso a terceros`
- **Descripcion:** contexto funcional + impacto.
- **Pasos de reproduccion:** copia exacta de esta guia.
- **Resultado actual:** evidencia (request/response/capturas).
- **Resultado esperado:** comportamiento seguro.
- **Impacto:** confidencialidad / integridad / disponibilidad.
- **Criterio de cierre:** prueba de regresion en staging.

---

Si quieres, en un segundo archivo te puedo generar una version **tipo checklist de QA** (mas corta) y otra **tipo pentest tecnico** (mas profunda con payloads y automatizacion).
