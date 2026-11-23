# Registro de seguridad · Sistema Alumnos

Estudiante: Renato Valenzuela   Repositorio: parcial3-ciberseguridad   Fecha: 23-11-2026

Completa este registro a medida que avanzas. Cada fila debe poder seguirse hasta un **build de Jenkins**
y un **commit** (usa el identificador `VULN-xx` en el mensaje del commit).

## 1. Revisiones de seguridad realizadas

| # | Etapa del SDLC | Herramienta / técnica | Build | Resultado (resumen) |
|---|---|---|---|---|
| R1 | Código (SAST) | Bandit en el pipeline | #2 → #6 | 5 hallazgos: 2 High / 2 Medium / 1 Low. Tras las correcciones: 0 hallazgos |
| R2 | Dependencias (SCA) | pip-audit en el pipeline | #2 → #6 | 19 vulnerabilidades en 4 paquetes (Flask, Werkzeug, Jinja2, gunicorn). Tras los PR de Dependabot: "No known vulnerabilities found" |
| R3 | Pruebas dinámicas (DAST) | OWASP ZAP sobre staging | #2 → #6 | 3 High / 3 Medium / 1 Low. Tras VULN-10: 0 High / 0 Medium / 0 Low (builds #5 y #6); solo 2 Informational |
| R4 | Revisión manual | Lectura de `app.py` | — | Fuga de la consulta SQL en el error 500, credenciales hardcodeadas y ausencia de límite de intentos de login |
| R5 | Producción | Grafana + Prometheus | — | Pendiente: datasource, dashboard y regla de alerta (evidencias 7 a 9) |

## 2. Vulnerabilidades identificadas y corregidas

| ID | Vulnerabilidad (CWE) | Detectada por | Ubicación | Riesgo | Corrección aplicada | Commit | Build que lo verifica |
|---|---|---|---|---|---|---|---|
| VULN-01 | XSS reflejado y DOM based en `/hello` (CWE-79) | ZAP | `app.py` · `hello()` | High | Escape del parámetro con `html.escape()` antes de devolver el HTML | `203eeaf` | #3 |
| VULN-02 | Inyección SQL en `/alumnos/buscar` (CWE-89) y fuga de la consulta en el error 500 (CWE-209) | Bandit B608 + ZAP + revisión manual | `app.py` · `buscar_alumnos()` | High | Consulta parametrizada `LIKE ?` y mensaje de error genérico | `986aa19` | #3 |
| VULN-03 | Contraseña con MD5 y credenciales de administrador hardcodeadas (CWE-327 / CWE-798) | Bandit B324 + revisión manual | `app.py` · `login()` | High | `generate_password_hash` / `check_password_hash` y credenciales inyectadas por entorno | `b97c2c3` | #3 |
| VULN-04 | `SECRET_KEY` hardcodeada (CWE-798) | Bandit B105 | `app.py` · configuración de Flask | Low | Lectura desde variable de entorno con valor aleatorio por defecto | `37862eb` | #3 |
| VULN-05 | `debug=True` expone el debugger de Werkzeug (CWE-94) | Bandit B201 | `app.py` · `app.run()` | High | `debug=False` | `0a56555` | #3 |
| VULN-06 | Bind a todas las interfaces `0.0.0.0` (CWE-605) | Bandit B104 | `app.py` · `app.run()` | Medium | Host configurable por entorno con default `127.0.0.1`; el bind de gunicorn en Docker no se modifica | `c4faea7` | #3 |
| VULN-07 | Ausencia de CSP, X-Frame-Options y X-Content-Type-Options (CWE-693) | ZAP | `app.py` · `despues_de_la_peticion()` | Medium | Cabeceras `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options` y `Referrer-Policy` en todas las respuestas | `52a1ac7` | #3 |
| VULN-08 | Ausencia de token anti-CSRF (CWE-352) | ZAP | `app.py` · `/` y `/login` | Medium | Token en sesión, campo oculto en el formulario y validación con `compare_digest` | `4c73dfc` | #3 |
| VULN-09 | Fuerza bruta sin limitación de intentos (CWE-307) | Revisión manual | `app.py` · `login()` | Medium | Límite de 5 intentos por IP en 60 s, respondiendo HTTP 429 | `27c49f2` | #3 |
| VULN-10 | CSP sin `form-action` y cookie de sesión sin `SameSite` (CWE-693 / CWE-1275) | ZAP | `app.py` · CSP y configuración de sesión | Medium | Directiva `form-action 'self'` y `SESSION_COOKIE_SAMESITE = 'Lax'` | `2cac8d3` | #5 |

(Agrega las filas que necesites.)

## 3. Resultados de OWASP ZAP

Escaneo sobre staging (`http://jenkins:5001`). Antes de las correcciones (build #2): **High=3, Medium=3, Low=1**. Después (build #6): **High=0, Medium=0, Low=0, Informational=2**. Las dos alertas Informational son informativas (detección de gestión de sesión y fuzzing de User-Agent), no requieren mitigación y no afectan el gate del pipeline.

| Alerta de ZAP | Riesgo | URL de ejemplo | VULN relacionada | Cómo se mitiga | ¿Mitigada? Build |
|---|---|---|---|---|---|
| XSS (DOM Based) | High | `/hello?name=` | VULN-01 | `html.escape()` del parámetro `name` antes de insertarlo en el HTML | Sí · #3 |
| XSS (Reflected) | High | `/hello?name=` | VULN-01 | `html.escape()` del parámetro `name` antes de insertarlo en el HTML | Sí · #3 |
| SQL Injection | High | `/alumnos/buscar?nombre=` | VULN-02 | Consulta parametrizada `LIKE ?` con el valor como parámetro, nunca concatenado | Sí · #3 |
| Absence of Anti-CSRF Tokens | Medium | `/` → `/login` | VULN-08 | Token aleatorio en sesión, campo oculto en el formulario y validación con `secrets.compare_digest` | Sí · #3 |
| CSP Header Not Set | Medium | `/` | VULN-07 | Cabecera `Content-Security-Policy` en `after_request` para todas las respuestas | Sí · #3 |
| Missing Anti-clickjacking Header | Medium | `/` | VULN-07 | Cabecera `X-Frame-Options: DENY` en `after_request` | Sí · #3 |
| X-Content-Type-Options Header Missing | Low | `/` | VULN-07 | Cabecera `X-Content-Type-Options: nosniff` en `after_request` | Sí · #3 |
| CSP: Failure to Define Directive with No Fallback | Medium | `/` | VULN-10 | Directiva `form-action 'self'` añadida a la CSP (no hereda de `default-src`) | Sí · #5 |
| Cookie without SameSite Attribute | Low | `/` | VULN-10 | `SESSION_COOKIE_SAMESITE = "Lax"` en la configuración de Flask | Sí · #5 |
| Session Management Response Identified | Informational | `/login` | — | Informativo: ZAP detecta una respuesta de gestión de sesión; sin acción requerida | No aplica · #6 |
| User Agent Fuzzer | Informational | `/` | — | Informativo: resultado del fuzzing de User-Agent; sin acción requerida | No aplica · #6 |

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
