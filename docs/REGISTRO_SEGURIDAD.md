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
| R5 | Producción | Grafana + Prometheus | — | Configurado: datasource `http://prometheus:9090`, dashboard `Producción · Sistema Alumnos` (4 paneles) y regla `IncidenteSeguridadProduccion` (umbral > 5). Detectó 2 ráfagas de ataque (ver sección 4) |

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

**Configuración:** datasource Prometheus (`http://prometheus:9090`), dashboard `Producción · Sistema Alumnos` con 4 paneles (`login_fallidos_total`, `peticiones_sospechosas_total` por `tipo`, `http_requests_total` por `status` y `app_info`) y regla de alerta `IncidenteSeguridadProduccion` con condición `WHEN last() OF A IS ABOVE 5`, evaluación cada 10 s y *Pending period: None* (disparo inmediato).

**Detección:** la regla se mantuvo en **Normal** mientras no hubo ataques (consulta = 0). Las dos ráfagas del script `scripts/simular_ataque.py` provocaron el paso a **Firing**; el contador acumulado pasó de 0 → 31 → 62 y volvió a Normal al salir el pico de la ventana de 1 minuto.

| Hora | Panel / métrica de Grafana | Qué se observó | Entorno | Clasificación | Acción de respuesta | Verificación |
|---|---|---|---|---|---|---|
| 2026-11-23 21:29:10 | `login_fallidos_total` (0 → 31), `peticiones_sospechosas_total{tipo="sqli"/"xss"}`, `http_requests_total{status="429"}` | 31 intentos de login en ~1 s; 1 patrón SQLi y 1 XSS detectados; mayoría de respuestas HTTP 429. Consulta de alerta = 31 (> 5) → **Firing** | Producción · `jenkins:5000` (job `produccion`) | Intento de fuerza bruta (CWE-307) + inyección SQL (CWE-89) + XSS reflejado (CWE-79) | Defensas ya desplegadas (build #7, `1.0.7+360f7f2`): rate limit 5 intentos/60 s → 429, consulta parametrizada, `html.escape()` y cabeceras CSP. Sin cambios de código durante el incidente | SQLi no devolvió datos internos; XSS escapado; `/health` y `/version` siguen OK; versión sin cambios; alerta volvió a Normal tras ~1 min |
| 2026-11-23 21:33:40 | `login_fallidos_total` (31 → 62) y `http_requests_total{status="429"}` | Segunda ráfaga de +31 intentos; la alerta volvió a **Firing** y el panel de login fallidos repitió el pico | Producción · `jenkins:5000` | Repetición del mismo ataque de laboratorio para verificar la recuperación de la alerta | Sin intervención manual: el rate limiting bloqueó automáticamente las peticiones | 429 sostenidos, servicio estable; la alerta volvió a Normal al expirar la ventana `[1m]`; build #7 sigue en SUCCESS |

**Resultado:** el monitoreo detectó ambos intentos, la clasificación fue correcta y las defensas respondieron sin intervención manual. No se requirió ningún commit ni despliegue nuevo: la versión en producción permaneció en `1.0.7+360f7f2`.

## 5. Gestión de dependencias

La configuración `.github/dependabot.yml` (ecosistema `pip`, directorio `/`, frecuencia semanal, límite 10 PRs y etiqueta `dependencias`) hizo que Dependabot detectara 19 vulnerabilidades en 4 paquetes y abriera los PRs #1 a #6. Todos fueron fusionados; el build #8 confirma que no se introdujeron vulnerabilidades nuevas.

| Paquete | Versión inicial | Vulnerabilidades (IDs) | Versión final | Origen del cambio (PR de Dependabot / manual) | ¿Pruebas, pip-audit y ZAP siguen OK? Build |
|---|---|---|---|---|---|
| Flask | 2.2.2 | PYSEC-2023-62, PYSEC-2026-2151 | 3.1.3 | PR #5 (`dependabot[bot]`) | Sí · #8 (9/9 pruebas, pip-audit 0 vuln., ZAP 0/0/0) |
| Werkzeug | 2.2.2 | PYSEC-2023-57, PYSEC-2023-58, PYSEC-2023-221, PYSEC-2026-2043, PYSEC-2026-2044, PYSEC-2026-2045, PYSEC-2026-2046, PYSEC-2026-2320, PYSEC-2026-3417, PYSEC-2026-4312 | 3.1.9 | PR #3 (`dependabot[bot]`) | Sí · #8 |
| Jinja2 | 3.1.2 | PYSEC-2026-1471, PYSEC-2026-1472, PYSEC-2026-1473, PYSEC-2026-1474, PYSEC-2026-1475 | 3.1.6 | PR #1 (`dependabot[bot]`) | Sí · #8 |
| gunicorn | 21.2.0 | PYSEC-2026-1433, PYSEC-2026-1434 | 27.0.1 | PR #6 (`dependabot[bot]`) | Sí · #8 |
| prometheus-client | 0.17.1 | Sin CVEs (actualización preventiva) | 0.26.0 | PR #2 (`dependabot[bot]`) | Sí · #8 |
| pytest (desarrollo) | 8.3.4 | Sin CVEs (actualización preventiva) | 9.2.0 | PR #4 (`dependabot[bot]`, `requirements-dev.txt`) | Sí · #8 |

**Cómo se aseguró que no hubiera nuevas vulnerabilidades:** cada PR se fusionó y el pipeline volvió a ejecutar las 9 pruebas funcionales, Bandit (0 hallazgos), pip-audit (*"No known vulnerabilities found"*) y OWASP ZAP sobre staging (0 High / 0 Medium / 0 Low). El build #6 fue el primero en verde tras los merges y el #8 lo confirmó de nuevo. Los conflicts de `requirements.txt` entre PRs se resolvieron conservando siempre la versión más alta de cada paquete.

## 6. Trazabilidad

El detalle completo, build por build, está en [`docs/TRAZABILIDAD.md`](TRAZABILIDAD.md) (copia del artifact `reports/TRAZABILIDAD.md` del build #8, generado por `scripts/trazabilidad.py`). Cada fila une build ↔ versión ↔ commit ↔ pruebas ↔ hallazgos ↔ despliegue.

| Actividad | Hallazgo | Build que lo detectó | Commit / PR que lo resolvió | Build que lo verificó | Versión en producción |
|---|---|---|---|---|---|
| Revisión de seguridad (Bandit) | 5 hallazgos (2 High / 2 Medium / 1 Low): B608, B324, B201, B104, B105 | #2 | Commits `203eeaf`…`2cac8d3` (VULN-01 a VULN-10) | #3 (Bandit 0/0/0) y #8 | 1.0.8+0035535 |
| Prueba automatizada (OWASP ZAP) | 7 alertas (3 High / 3 Medium / 1 Low) + 2 nuevas tras la primera corrección | #2 y #3 | VULN-01, VULN-02, VULN-07, VULN-08, VULN-10 | #5 (0/0/0) y #8 | 1.0.8+0035535 |
| Gestión de dependencias (Dependabot) | 19 vulnerabilidades en 4 paquetes (Flask, Werkzeug, Jinja2, gunicorn) | #2 | PRs #1 a #6 de `dependabot[bot]` | #6 (0 vuln., primer SUCCESS) y #8 | 1.0.8+0035535 |
| Monitoreo (Grafana + Prometheus) | 2 ráfagas de ataque en producción (21:29:10 y 21:33:40) | Alerta `IncidenteSeguridadProduccion` | Defensas ya desplegadas: rate limit, consulta parametrizada y escape HTML (sin commit nuevo) | #8 (SUCCESS, versión sin cambios) | 1.0.8+0035535 |

**Recorrido completo del SDLC:** el build #2 dejó constancia del código vulnerable (Bandit 2/2/1, pip-audit 19 vuln., ZAP 3/3/1) y bloqueó el despliegue a producción. Las correcciones `VULN-01`…`VULN-10` y los PRs de Dependabot se fueron integrando en los builds #3 a #6, hasta alcanzar el primer pipeline verde con despliegue seguro. Los builds #7 y #8 añadieron la documentación y el monitoreo sin alterar el código desplegado. El resultado es trazable desde cada hallazgo hasta su commit/PR, su build verificador y la versión final en producción.
