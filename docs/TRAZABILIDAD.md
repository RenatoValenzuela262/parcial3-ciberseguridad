# Registro de trazabilidad - Sistema Alumnos

Cada fila la agrega Jenkins al terminar un build (`scripts/trazabilidad.py`).

Copia del artifact `reports/TRAZABILIDAD.md` del **build #8** (el resultado con el monitoreo documentado). El registro original se actualiza en cada build en `/var/jenkins_home/registro-trazabilidad/TRAZABILIDAD.md`.

| Build | Fecha (UTC) | Versión | Commit | Mensaje del commit | Pruebas | Bandit H/M/L | pip-audit | ZAP H/M/L | Resultado | Producción |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 2026-11-23 20:30 | 1.0.1+c658ed8 | `c658ed8` | Carpeta proyecto | 9/9 OK | n/a | n/a | n/a | SUCCESS | 1.0.1+c658ed8 |
| 2 | 2026-11-23 21:33 | 1.0.2+a74bee0 | `a74bee0` | CI: agrega etapas de seguridad (SAST, SCA, ZAP) y Dependabot | 9/9 OK | 2/2/1 | 19 vuln. | 3/3/1 | UNSTABLE | no desplegado |
| 3 | 2026-11-23 22:03 | 1.0.3+27c49f2 | `27c49f2` | VULN-09: limitacion de intentos de login por IP (CWE-307) | 9/9 OK | 0/0/0 | 19 vuln. | 0/1/1 | UNSTABLE | no desplegado |
| 4 | 2026-11-23 22:29 | 1.0.4+c53a4b1 | `c53a4b1` | Actualizacion Resitro de seguridad (info alumno) | 9/9 OK | 0/0/0 | 19 vuln. | 0/1/1 | UNSTABLE | no desplegado |
| 5 | 2026-11-23 22:40 | 1.0.5+21b5993 | `21b5993` | Doc: completa registro de seguridad secciones 1 y 2 (evidencia 4) | 9/9 OK | 0/0/0 | 19 vuln. | 0/0/0 | UNSTABLE | no desplegado |
| 6 | 2026-11-23 22:58 | 1.0.6+8c950d0 | `8c950d0` | Merge PR #4 | 9/9 OK | 0/0/0 | 0 vuln. | 0/0/0 | SUCCESS | 1.0.6+8c950d0 |
| 7 | 2026-11-23 23:20 | 1.0.7+360f7f2 | `360f7f2` | Doc: seccion 3 con resultados ZAP antes/despues y ajuste de builds (ev | 9/9 OK | 0/0/0 | 0 vuln. | 0/0/0 | SUCCESS | 1.0.7+360f7f2 |
| 8 | 2026-11-24 00:43 | 1.0.8+0035535 | `0035535` | Doc: seccion 4 con monitoreo e incidentes detectados en Grafana (evide | 9/9 OK | 0/0/0 | 0 vuln. | 0/0/0 | SUCCESS | 1.0.8+0035535 |
