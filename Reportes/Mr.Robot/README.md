# Reporte Red/Blue — Mr. Robot (HTB)

**Autor:** Diego Vásquez (dvasquez-design)
**Target:** `10.10.13.11` (HTB — Mr. Robot)
**Fecha de la explotación:** registrada en [`pentestlab-cli-walkthrough.md`](https://github.com/dvasquez-design/Cloudlab/blob/main/PentestLab/pentestlab-cli-walkthrough.md)
**Fecha del escaneo DAST (ZAP):** 11 oct 2026
**Documento 1 de 4** de la serie de reportes de correlación Red/Blue sobre el toolkit CLI de PentestLab.

---

## 0. Qué es este documento

Este reporte toma lo que se explotó manualmente del lado ofensivo y lo contrasta con lo que un escáner automático detectó — o no detectó — por su cuenta. El objetivo es doble:

1. **Medir la cobertura del escáner automático** frente a una explotación manual real y documentada.
2. **Clasificar cada hallazgo con un criterio de seguridad formal** (principios de la tríada CIA, control de acceso, autenticación, gestión de riesgos), para que el reporte no sea solo "esto se rompió" sino "esto se rompió, por qué principio de seguridad falló, y qué control lo hubiera prevenido".

Dos fuentes alimentan este reporte:

- **Lado red (explotación manual, 100% CLI):** la cadena de ataque completa contra Mr. Robot, documentada comando por comando en [`pentestlab-cli-walkthrough.md`](https://github.com/dvasquez-design/Cloudlab/blob/main/PentestLab/pentestlab-cli-walkthrough.md), dentro del repositorio [CloudLab/PentestLab](https://github.com/dvasquez-design/Cloudlab/tree/main/PentestLab).
- **Lado blue (DAST automático):** escaneo pasivo/activo con **OWASP ZAP 2.17.0** contra el mismo target — ver `ZAP-Scan-Report.pdf` en esta misma carpeta.

Cada hallazgo se clasifica según los principios básicos de seguridad de la información (confidencialidad, integridad, disponibilidad, gestión de riesgos, modelos de autenticación y control de acceso) — el mismo marco conceptual que cubren certificaciones como ISC2 CC o CompTIA Security+, usado acá como criterio de análisis, no como fuente a citar.

> **Nota de alcance:** el reporte de ZAP incluye entradas de `firefox-settings-attachments.cdn.mozilla.net` y `oimg.nbcuni.com`. Son ruido de fondo capturado por el proxy durante la sesión de navegación (telemetría de Firefox y un beacon de analítica embebido en la propia página de Mr. Robot), no objetivos del engagement. El análisis de este reporte se limita a `10.10.13.11`.

---

## 1. Resumen ejecutivo

| | |
|---|---|
| **Compromiso total** | Sí — RCE → `www-data`/`daemon` → `robot` → `root` |
| **Vector inicial** | Enumeración de usuario WordPress por respuesta diferencial + fuerza bruta de contraseña contra `wp-login.php` |
| **Hallazgos DAST (ZAP)** | 0 Críticos · 2 Medios · 3 Bajos · 4 Informativos |
| **Brecha principal entre ambos lados** | El escáner automático **no detectó** la ruta de explotación real (enumeración de usuario + fuerza bruta + RCE vía Theme Editor). Encontró síntomas correlacionados (componente desactualizado, falta de cabeceras) pero no la cadena de ataque completa — ver sección 4. |
| **Causa raíz de mayor impacto** | Componentes desactualizados sin ciclo de parcheo: PHP 5.5.29 (EOL desde 2016) + jQuery 2.1.3/2.1.4 (vulnerable, CVE-2019-11358 entre otros), combinados con una superficie de administración (Theme Editor) que permite RCE por diseño a cualquier cuenta admin. |

---

## 2. Lado red — cadena de ataque (resumen)

Cadena completa, comando por comando, en [`pentestlab-cli-walkthrough.md`](https://github.com/dvasquez-design/Cloudlab/blob/main/PentestLab/pentestlab-cli-walkthrough.md) (sección 3). Resumen de fases para este reporte:

| Fase | Acción | Resultado |
|---|---|---|
| 0 — Recon | `nmap -sV -sC -p-`, `httpx`, `enum4linux-ng` | Servicio web HTTP/HTTPS, sin superficie SMB adicional |
| 1 — Enumeración de contenido | `gobuster dir`, `robots.txt` | `fsocity.dic` + `key-1-of-3.txt` expuestos sin control de acceso → **Key 1** |
| 2 — Enumeración de usuario | `ffuf` contra `wp-login.php`, filtrando por mensaje de error diferencial | Usuario válido confirmado: `Elliot` |
| 3 — Fuerza bruta de contraseña | `wpscan -U Elliot -P fsocity_clean.dic` | Credenciales: `Elliot / ER28-0652` (sin MFA, sin bloqueo de cuenta) |
| 4 — RCE | Login admin → WordPress Theme Editor → PHP malicioso en `404.php` → reverse shell | Shell como `daemon` |
| 5 — Pivote | Hash MD5 de `robot` crackeado con `john` (mismo wordlist) | **Key 2**, usuario `robot` |
| 6 — Escalación | `nmap --interactive` (binario SUID antiguo) → `!sh` | **Key 3**, root confirmado |

**Hallazgo post-explotación relevante:** `wp-config.php` contenía credenciales de base de datos y FTP en texto plano — reutilizadas entre servicios.

---

## 3. Lado blue — hallazgos DAST (ZAP) y correlación con el lado red

Tabla de correlación: cada fila conecta el hallazgo de ZAP con la fase de explotación manual que lo explica (si existe) y el principio de seguridad que clasifica el hallazgo.

| Hallazgo ZAP | Riesgo | CWE | ¿Lo explotó el lado red? | Principio de seguridad / control afectado |
|---|---|---|---|---|
| **Vulnerable JS Library** (jQuery 2.1.3 / 2.1.4, CVE-2019-11358 y otros) | Medio | CWE-1395 | Indirecto — evidencia de que la pila de la app no recibe parches | Gestión de riesgos: componente sin mitigación aplicada = riesgo inherente no tratado |
| **CSP Header Not Set** | Medio | CWE-693 | No explotado en esta corrida, pero habilita XSS persistente si se combina con el Theme Editor | Integridad — ausencia de control compensatorio de navegador |
| **Server Leaks Info — `X-Powered-By: PHP/5.5.29`** | Bajo | CWE-497 | Confirma la versión de PHP que permitió la cadena de RCE (EOL, sin soporte de seguridad desde 2016) | Confidencialidad (fuga de información de huella técnica) |
| **Strict-Transport-Security Header Not Set** | Bajo | CWE-319 | No explotado directamente, pero el login ocurrió sin forzar HTTPS en todas las rutas | Criptografía práctica — datos en tránsito sin TLS forzado |
| **X-Content-Type-Options Header Missing** (sistémico) | Bajo | CWE-693 | No explotado | Integridad — falta de control compensatorio anti-MIME-sniffing |
| **Information Disclosure — Suspicious Comments** (3 instancias) | Info | CWE-615 | Relacionado al mismo principio que `robots.txt` exponiendo `key-1-of-3.txt` | Confidencialidad — exposición de artefactos de desarrollo/depuración |
| **Session Management Response Identified** | Info | — | Confirma el mecanismo de cookies que se reutilizó manualmente (`-c cookies.txt` / `-b cookies.txt`) | Modelo AAA — gestión de sesión sin flags de seguridad visibles en el hallazgo |

### 4. La brecha real: lo que ZAP no vio

Este es el valor central de cruzar ambos lados. Un DAST no autenticado, corrido sin credenciales ni flujo de negocio conocido, **no reprodujo la cadena de ataque real**:

- No detectó la **enumeración de usuario por respuesta diferencial** en `wp-login.php` — requiere lógica de negocio (comparar dos respuestas con nombre de usuario válido vs. inválido), algo que un passive/active scanner estándar sin reglas específicas de WordPress no cubre.
- No detectó la **ausencia de MFA / bloqueo de cuenta** que permitió la fuerza bruta exitosa con `wpscan` — este tipo de control ausente no es visible en una sola petición HTTP, requiere múltiples intentos coordinados y conocimiento del contexto de autenticación.
- No marcó el **WordPress Theme Editor** como superficie de RCE — es una funcionalidad legítima del CMS para administradores, no una vulnerabilidad per se; el riesgo es de **modelo de confianza** (cualquier admin puede escribir PHP arbitrario), no un bug de código — motivo por el cual un scanner de vulnerabilidades no lo señala.
- No evaluó el **post-explotación** (SUID de `nmap`, credenciales en `wp-config.php`): son hallazgos de una fase a la que un DAST externo, por definición, no llega.

**Conclusión:** ZAP cubrió bien la capa de higiene HTTP (cabeceras, componentes desactualizados, fuga de banner) pero cero cobertura sobre la cadena de autenticación y post-explotación. Esto confirma el valor de combinar DAST con explotación manual — y es exactamente la comparación que ya documenta [`burpsuite-zap-cli-clon.md`](https://github.com/dvasquez-design/Cloudlab/blob/main/PentestLab/burpsuite-zap-cli-clon.md) desde el ángulo de herramientas CLI vs. GUI.

---

## 5. Recomendaciones priorizadas

| Prioridad | Recomendación | Hallazgo que mitiga |
|---|---|---|
| 1 | Actualizar PHP (5.5.29 → versión soportada) y jQuery (≥3.5.0) | Vulnerable JS Library, X-Powered-By leak |
| 2 | Forzar MFA y bloqueo de cuenta tras N intentos fallidos en `wp-login.php` | Vector de explotación completo (fases 2–4) |
| 3 | Restringir o auditar el uso del Theme Editor (deshabilitar edición de archivos vía `DISALLOW_FILE_EDIT` en `wp-config.php`) | RCE vía Theme Editor |
| 4 | Agregar cabeceras de seguridad: `Content-Security-Policy`, `Strict-Transport-Security`, `X-Content-Type-Options: nosniff` | 3 hallazgos Bajos de ZAP |
| 5 | Eliminar archivos sensibles accesibles desde `robots.txt` (`key-*.txt`, wordlists) | Exposición inicial (Key 1) |
| 6 | No reutilizar credenciales entre DB/FTP/app; mover secretos fuera de `wp-config.php` en texto plano | Hallazgo post-explotación |

---

## 6. Referencias

- Walkthrough técnico completo (lado red, CLI): [`pentestlab-cli-walkthrough.md`](https://github.com/dvasquez-design/Cloudlab/blob/main/PentestLab/pentestlab-cli-walkthrough.md)
- Comparación de stack CLI vs. Burp Suite vs. ZAP: [`burpsuite-zap-cli-clon.md`](https://github.com/dvasquez-design/Cloudlab/blob/main/PentestLab/burpsuite-zap-cli-clon.md)
- Reporte DAST original: `ZAP-Scan-Report.pdf` (esta carpeta)
- [OWASP ZAP — Burp to ZAP feature map](https://www.zaproxy.org/docs/burp-to-zap-feature-map/)
- CWE-1395, CWE-693, CWE-497, CWE-319, CWE-615 — [cwe.mitre.org](https://cwe.mitre.org/)
