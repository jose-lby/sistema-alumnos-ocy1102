# Registro de seguridad · Sistema Alumnos

Estudiante: ____________________   Repositorio: ____________________   Fecha: __________

Completa este registro a medida que avanzas. Cada fila debe poder seguirse hasta un **build de Jenkins**
y un **commit** (usa el identificador `VULN-xx` en el mensaje del commit).

# Registro de seguridad · Sistema Alumnos

Estudiante: José Luis Lebuy Narváez   Repositorio: https://github.com/jose-lby/sistema-alumnos-ocy1102   Fecha: 2026-11-27

## 1. Revisiones de seguridad realizadas

| # | Etapa del SDLC | Herramienta / técnica | Build | Resultado (resumen) |
|---|---|---|---|---|
| R1 | Código (SAST) | Bandit en el pipeline | #3 (antes) / #7 (después) | 2 High / 2 Medium / 1 Low → 0 High / 0 Medium / 0 Low |
| R2 | Dependencias (SCA) | pip-audit en el pipeline | #3 | 19 vulnerabilidades en 4 paquetes (Flask, Werkzeug, Jinja2, gunicorn) |
| R3 | Pruebas dinámicas (DAST) | OWASP ZAP sobre staging | #3 (antes) / #4 (después) | 3 High / 3 Medium / 1 Low → 0 High / 0 Medium / 0 Low |
| R4 | Revisión manual | Lectura de `app.py` | — | Se detectó XSS en `/hello`, fuga de consulta SQL en errores 500, ausencia de CSRF y falta de límite de intentos en `/login` |
| R5 | Producción | Grafana + Prometheus | — | Pendiente de ejecución en producción (sección 4) |

## 2. Vulnerabilidades identificadas y corregidas

| ID | Vulnerabilidad (CWE) | Detectada por | Ubicación | Riesgo | Corrección aplicada | Commit | Build que lo verifica |
|---|---|---|---|---|---|---|---|
| VULN-01 | Inyección SQL y exposición de errores SQL (CWE-89 / CWE-209) | Bandit (B608), OWASP ZAP, revisión manual | `app.py:205` (`/alumnos/buscar`) | Alto | Se reemplazó la concatenación de strings por consulta SQL parametrizada (`?`) y se ocultó el detalle técnico del error en respuestas HTTP 500. | `<hash>` | #4 |
| VULN-02 | Cross-Site Scripting reflejado y basado en DOM (CWE-79) | OWASP ZAP, revisión manual | `app.py:165` (`/hello`) | Alto | Se aplicó `markupsafe.escape()` al parámetro `name` y se estructuró una respuesta HTML válida con codificación UTF-8. | `<hash>` | #4 |
| VULN-03 | Secretos hardcodeados, hash criptográfico débil MD5 y falta de protección contra fuerza bruta (CWE-259 / CWE-327 / CWE-307) | Bandit (B105, B324), revisión manual | `app.py:20-22` y `app.py:221` (`/login`) | Alto | Se eliminaron `SECRET_KEY` y el hash MD5 fijo; se leen desde variables de entorno con respaldo seguro (`secrets.token_hex`), se migró a `werkzeug.security.check_password_hash` y se agregó límite de 5 intentos por minuto por IP (HTTP 429). | `<hash>` | #4 |
| VULN-04 | Ausencia de tokens Anti-CSRF y banderas seguras de sesión (CWE-352) | OWASP ZAP, revisión manual | `app.py` (`PORTADA` y `/login`) | Medio | Se generaron tokens CSRF aleatorios por sesión (`csrf_token`), se validan con `hmac.compare_digest` (HTTP 403 si falla) y se activaron `SESSION_COOKIE_HTTPONLY=True` y `SESSION_COOKIE_SAMESITE="Lax"`. | `<hash>` | #4 |
| VULN-05 | Ausencia de cabeceras HTTP de seguridad (CWE-693 / CWE-1021) | OWASP ZAP (escaneo pasivo) | `app.py` (`despues_de_la_peticion`) | Medio | Se agregaron `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy` y `Permissions-Policy` en todas las respuestas. | `<hash>` | #4 |
| VULN-06 | Modo `debug=True` activo y escucha abierta en `0.0.0.0` (CWE-94 / CWE-605) | Bandit (B201, B104) | `app.py:241` (`__main__`) | Alto | Se fijó `debug=False` y se restringió el host por defecto a `127.0.0.1` configurable por variable de entorno. | `<hash>` | #4 |



## 3. Resultados de OWASP ZAP

| Alerta de ZAP | Riesgo | URL de ejemplo | VULN relacionada | Cómo se mitiga | ¿Mitigada? Build |
|---|---|---|---|---|---|
| Cross Site Scripting (DOM Based) | High | `http://jenkins:5001/hello?name=%3Cscript%3Ealert(5397)%3C/script%3E` | VULN-02 | Escape del parámetro `name` con `markupsafe.escape()` antes de insertarlo en HTML. | Sí · #4 |
| Cross Site Scripting (Reflected) | High | `http://jenkins:5001/hello?name=%3C%2Fh1%3E%3CscrIpt%3Ealert%281%29%3B%3C%2FscRipt%3E%3Ch1%3E` | VULN-02 | Escape HTML estricto de la salida en `/hello` y CSP restrictiva (`script-src 'self'`). | Sí · #4 |
| SQL Injection | High | `http://jenkins:5001/alumnos/buscar?nombre=%27` | VULN-01 | Consultas SQL parametrizadas con marcadores `?` en SQLite y control seguro de errores. | Sí · #4 |
| Absence of Anti-CSRF Tokens | Medium | `http://jenkins:5001` | VULN-04 | Token aleatorio `csrf_token` por sesión en el formulario `/login` y verificación con `hmac.compare_digest`. | Sí · #4 |
| Content Security Policy (CSP) Header Not Set | Medium | `http://jenkins:5001` | VULN-05 | Cabecera `Content-Security-Policy` restrictiva en `@app.after_request`. | Sí · #4 |
| Missing Anti-clickjacking Header | Medium | `http://jenkins:5001` | VULN-05 | Cabecera `X-Frame-Options: DENY` y directiva `frame-ancestors 'none'` en CSP. | Sí · #4 |
| X-Content-Type-Options Header Missing | Low | `http://jenkins:5001` | VULN-05 | Cabecera `X-Content-Type-Options: nosniff` en todas las respuestas HTTP. | Sí · #4 |
## 4. Monitoreo e incidentes

| Hora | Panel / métrica de Grafana | Qué se observó | Entorno | Clasificación | Acción de respuesta | Verificación |
|---|---|---|---|---|---|---|
| | | | | | | |

## 5. Gestión de dependencias

| Paquete | Versión inicial | Vulnerabilidades (IDs) | Versión final | Origen del cambio (PR de Dependabot / manual) | ¿Pruebas, pip-audit y ZAP siguen OK? Build |
|---|---|---|---|---|---|
| Flask | 2.2.2 | | | | |
| Werkzeug | 2.2.2 | | | | |
| Jinja2 | 3.1.2 | | | | |
| gunicorn | 21.2.0 | | | | |
| prometheus-client | 0.17.1 | | | | |

## 6. Trazabilidad

Pega aquí (o adjunta) el `TRAZABILIDAD.md` del último build y completa un recorrido por cada actividad del pipeline:

| Actividad | Hallazgo | Build que lo detectó | Commit / PR que lo resolvió | Build que lo verificó | Versión en producción |
|---|---|---|---|---|---|
| Revisión de seguridad (Bandit) | | #__ | | #__ | |
| Prueba automatizada (OWASP ZAP) | | #__ | | #__ | |
| Gestión de dependencias (Dependabot) | | #__ | PR #__ | #__ | |
