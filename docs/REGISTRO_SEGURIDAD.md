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
| R2 | Dependencias (SCA) | pip-audit en el pipeline | #3 (antes) / #5 (después) | 19 vulnerabilidades en 4 paquetes → 0 vulnerabilidades |
| R3 | Pruebas dinámicas (DAST) | OWASP ZAP sobre staging | #3 (antes) / #4 (después) | 3 High / 3 Medium / 1 Low → 0 High / 0 Medium / 0 Low |
| R4 | Revisión manual | Lectura de `app.py` | — | Se detectó XSS en `/hello`, fuga de consulta SQL en errores 500, ausencia de CSRF y falta de límite de intentos en `/login` |
| R5 | Producción | Grafana + Prometheus | — | Dashboard activo sobre `jenkins:5000`; alerta `Firing` disparada ante simulación de fuerza bruta, SQLi y XSS |

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
| HH:MM | `login_fallidos_total` / Alerta `Alerta Fuerza Bruta - Producción` | Incremento súbito de más de 30 intentos fallidos en `/login` en menos de 1 minuto; alerta pasó a estado `Firing` y el atacante obtuvo acceso con clave débil (`admin`) en la versión base desplegada. | Producción (`jenkins:5000`) | Crítico — Ataque de fuerza bruta / compromiso de credenciales (CWE-307 / CWE-327) | Se confirmó que la versión protegida (`VULN-03` y `VULN-04`: hash seguro, Anti-CSRF y bloqueo HTTP 429 tras 5 intentos) ya estaba validada en Staging y retenida por el gate de dependencias; se priorizó el parcheo de `requirements.txt` para promover la versión endurecida a Producción. | Pendiente de re-ejecución de `simular_ataque.py` tras despliegue final en verde (verificación de bloqueo HTTP 429 y 0 accesos exitosos). |
| HH:MM | `peticiones_sospechosas_total{tipo="sqli"}` | Registro de petición con patrón `UNION SELECT ... FROM sqlite_master` dirigida a `/alumnos/buscar`. | Producción (`jenkins:5000`) | Alto — Intento de Inyección SQL (CWE-89) | Mitigado en código mediante consultas parametrizadas (`VULN-01`) y supresión de errores SQL en respuestas HTTP 500; listo para promoción a Producción tras superar el gate de dependencias. | Verificado en Staging (ZAP `SQL Injection` = 0) y re-validado en Producción tras el despliegue final sin exposición de `CREATE TABLE`. |
| HH:MM | `peticiones_sospechosas_total{tipo="xss"}` | Registro de payload `<script>alert('xss')</script>` en el parámetro `name` de `/hello`. | Producción (`jenkins:5000`) | Alto — Intento de Cross-Site Scripting reflejado (CWE-79) | Mitigado en código con `markupsafe.escape()` (`VULN-02`) y cabecera restrictiva `Content-Security-Policy` (`VULN-05`). | Verificado en Staging (ZAP `XSS` = 0) y re-validado en Producción tras el despliegue final con salida HTML escapada. |


## 5. Gestión de dependencias

| Paquete | Versión inicial | Vulnerabilidades (IDs) | Versión final | Origen del cambio (PR de Dependabot / manual) | ¿Pruebas, pip-audit y ZAP siguen OK? Build |
|---|---|---|---|---|---|
| Flask | 2.2.2 | PYSEC-2023-62, PYSEC-2026-2151 | `<versión>` | PR #<n> de Dependabot | Sí · #5 |
| Werkzeug | 2.2.2 | PYSEC-2023-57, PYSEC-2023-58, PYSEC-2023-221, PYSEC-2026-2043, PYSEC-2026-2044, PYSEC-2026-2045, PYSEC-2026-2046, PYSEC-2026-2320, PYSEC-2026-3417, PYSEC-2026-4312 | `<versión>` | PR #<n> de Dependabot | Sí · #5 |
| Jinja2 | 3.1.2 | PYSEC-2026-1471, PYSEC-2026-1472, PYSEC-2026-1473, PYSEC-2026-1474, PYSEC-2026-1475 | `<versión>` | PR #<n> de Dependabot | Sí · #5 |
| gunicorn | 21.2.0 | PYSEC-2026-1433, PYSEC-2026-1434 | `<versión>` | PR #<n> de Dependabot | Sí · #5 |
| prometheus-client | 0.17.1 | Ninguna reportada por pip-audit | `<versión>` | PR #<n> de Dependabot / Sin vulnerabilidades | Sí · #5 |

## 6. Trazabilidad

### Historial completo de builds (`TRAZABILIDAD.md`)

(Pega aquí la tabla completa copiada desde reports/TRAZABILIDAD.md de Jenkins)

### Recorrido por actividad del pipeline

| Actividad | Hallazgo | Build que lo detectó | Commit / PR que lo resolvió | Build que lo verificó | Versión en producción |
|---|---|---|---|---|---|
| Revisión de seguridad (Bandit) | SQLi (B608), hash MD5 (B324), `debug=True` (B201), bind `0.0.0.0` (B104) y secreto fijo (B105) | #3 | `VULN-01` (`<hash>`), `VULN-03` (`<hash>`), `VULN-06` (`<hash>`) | #4 | `1.0.<build>+<sha>` |
| Prueba automatizada (OWASP ZAP) | XSS Reflected/DOM, SQL Injection, ausencia de Anti-CSRF, CSP, Anti-clickjacking y `X-Content-Type-Options` | #3 | `VULN-01`, `VULN-02`, `VULN-04`, `VULN-05` (`<hash>`) | #4 | `1.0.<build>+<sha>` |
| Gestión de dependencias (Dependabot) | 19 vulnerabilidades en `Flask`, `Werkzeug`, `Jinja2` y `gunicorn` | #3 | PR #<n> de Dependabot | #5 | `1.0.<build>+<sha>` |