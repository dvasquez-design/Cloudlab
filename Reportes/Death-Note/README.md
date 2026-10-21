# Reporte Red/Blue — Death Note (homelab)

**Target:** `deathnote.vuln` (`10.10.13.41` — homelab `pentest-lab`, libvirt)
**Fecha de la explotación:** registrada en [`sesion-pentestlab-deathnote.md`](./sesion-pentestlab-deathnote.md)
**Fecha del escaneo DAST (ZAP):** 18 oct 2026
**Documento 2 de la serie de reportes de correlación Red/Blue sobre el toolkit CLI de PentestLab.**

---

## 0. Qué es este documento

Este reporte toma lo que se explotó manualmente del lado ofensivo y lo contrasta con lo que un escáner automático detectó — o no detectó — por su cuenta. El objetivo es doble:

1. **Medir la cobertura del escáner automático** frente a una explotación manual real y documentada.
2. **Clasificar cada hallazgo con un criterio de seguridad formal** (principios de la tríada CIA, control de acceso, gestión de riesgos), para que el reporte no sea solo "esto se rompió" sino "esto se rompió, por qué principio de seguridad falló, y qué control lo hubiera prevenido".

Dos fuentes alimentan este reporte:

- **Lado red (explotación manual):** la cadena de ataque completa contra Death Note, documentada paso a paso en [`sesion-pentestlab-deathnote.md`](./sesion-pentestlab-deathnote.md).
- **Lado blue (DAST automático):** escaneo con **OWASP ZAP 2.17.0** contra el mismo target — ver `ZAP-Scan-Report.pdf` en esta misma carpeta.

Cada hallazgo se clasifica según los principios básicos de seguridad de la información: confidencialidad, integridad, disponibilidad, gestión de riesgos y control de acceso.

---

## 1. Resumen ejecutivo

| | |
|---|---|
| **Compromiso total** | Sí — Directory Listing → notas filtradas → fuerza bruta SSH → shell `l` → enumeración post-login → password filtrada en `case.wav` → `su kira` → `sudo` sin restricciones → root |
| **Vector inicial** | Directory listing habilitado en `wp-content/uploads/2021/07/` expone `notes.txt` (wordlist) y `user.txt` (lista de usuarios) sin control de acceso |
| **Hallazgos DAST (ZAP)** | 1 High · 5 Medium · 4 Low · 4 Informativos |
| **Brecha principal entre ambos lados** | El escáner detectó el síntoma correcto (Directory Browsing) pero no pudo encadenarlo con la fuerza bruta SSH ni con el paso a root posterior — eso requirió explotación manual completa. Además, el único hallazgo High del ZAP (SQL Injection) quedó **sin cerrar**: dos intentos de verificación manual con `sqlmap` no lo confirmaron, pero el análisis más profundo (el más agresivo de ambos) no se completó — ver sección 4. |
| **Causa raíz de mayor impacto** | Credenciales débiles y reutilizables expuestas por una mala configuración de servidor (`Options +Indexes`), combinadas con ausencia de MFA/bloqueo de cuenta en SSH y un modelo de confianza de sudo sin restricciones para `kira`. |

---

## 2. Lado red — cadena de ataque (resumen)

Cadena completa, paso a paso, en [`sesion-pentestlab-deathnote.md`](./sesion-pentestlab-deathnote.md) (sección 1). Resumen de fases para este reporte:

| Fase | Acción | Resultado |
|---|---|---|
| 0 — Recon | `nmap -A -T4` | 22/ssh (OpenSSH 7.9p1 Debian), 80/http (Apache 2.4.38) |
| 1 — Hints | `curl` al index, `robots.txt`, `important.jpg` | Meta-refresh hacia `/wordpress`; hint narrativo apunta a `user.txt` como login |
| 2 — Enumeración de contenido | `gobuster dir` contra `/wordpress/` | Confirma rutas estándar de WP (`wp-content`, `wp-admin`, `wp-includes`) |
| 3 — Directory listing | `nikto` (parcial) + `curl` directo sobre `wp-content/uploads/2021/07/` | `notes.txt` (wordlist) + `user.txt` (usuarios) expuestos sin control de acceso |
| 4 — Fuerza bruta SSH | `hydra -L user.txt -P notes.txt ... ssh` | Credenciales: `l / death4me` (sin MFA, sin bloqueo de cuenta) |
| 5 — Post-login | Decodificación Brainfuck de `user.txt` | Mensaje narrativo de "L" (no flag tradicional) |
| 5.5 — Enumeración de privilegios | `sudo -l`, exploración de `/home` y `/opt` | `l` sin sudo; `/opt/L/fake-notebook-rule/case.wav` hallado (ruta distinta a la del walkthrough de referencia) |
| 6 — Credencial oculta | `cat case.wav \| xxd -r -p \| base64 -d` | Password: `kiraisevil` |
| 7 — De `kira` a root | `su kira` → `sudo cat /root/root.txt` | Root confirmado — `kira` con sudo sin restricciones |

**Hallazgo post-explotación relevante:** la contraseña de `kira` estaba oculta en un archivo de audio (esteganografía simple hex→base64) en vez de en texto plano — técnica de ofuscación básica, no un control de seguridad real.

---

## 3. Lado blue — hallazgos DAST (ZAP) y correlación con el lado red

| Hallazgo ZAP | Riesgo | CWE | ¿Lo explotó el lado red? | Principio de seguridad / control afectado |
|---|---|---|---|---|
| **SQL Injection** (`wp-comments-post.php`, `comment_post_ID`) | High | CWE-89 | **Sin cerrar** — `sqlmap` a nivel 3/riesgo 2 no confirmó inyección; un segundo intento más agresivo (nivel 5/riesgo 3) quedó sin resultado final (ver sección 4) | Validación de entrada / consultas no parametrizadas — requiere análisis más profundo |
| **Directory Browsing** (systemic — `wp-content/uploads/2021/07/`, temas) | Medium | CWE-548 | **Sí** — es el vector de acceso inicial completo de la cadena (notes.txt + user.txt) | Control de acceso — configuración de servidor (`Options +Indexes`) sin restricción |
| **Absence of Anti-CSRF Tokens** (form de comentarios, 2 instancias) | Medium | CWE-352 | No explotado — mismo form del SQLi, sin protección adicional | Integridad — ausencia de control contra solicitudes forjadas |
| **CSP Header Not Set** (systemic) | Medium | CWE-693 | No explotado, pero habilitaría XSS persistente si se combina con otro vector de inyección | Integridad — falta de control compensatorio de navegador |
| **Missing Anti-clickjacking Header** (systemic) | Medium | CWE-1021 | No explotado | Integridad — sin protección contra framing |
| **HTTP Only Site** | Medium | CWE-311 | No explotado directamente, pero toda la sesión (incluida la fuerza bruta SSH) ocurrió sin cifrado de transporte en la capa web | Criptografía práctica — datos en tránsito sin TLS |
| **Server Leaks Version Information** (Apache 2.4.38, systemic) | Low | CWE-497 | Confirma la versión de Apache usada durante toda la explotación | Confidencialidad — fuga de huella técnica |
| **Private IP Disclosure** (192.168.1.54, 7 instancias — feeds RSS / REST API de WP) | Low | CWE-497 | No explotado, pero es info de red interna útil para un atacante con más tiempo | Confidencialidad — exposición de topología interna |
| **X-Content-Type-Options Header Missing** (systemic) | Low | CWE-693 | No explotado | Integridad — falta de control compensatorio anti-MIME-sniffing |
| **In Page Banner Information Leak** (`sitemap.xml`) | Low | CWE-497 | Relacionado al mismo principio que el banner de Apache | Confidencialidad |
| **User Controllable HTML Element Attribute (Potential XSS)** (`replytocom`) | Info | CWE-20 | No verificado (pendiente, verificación barata no realizada en esta sesión) | Validación de entrada — hot-spot sin confirmar explotabilidad |
| **Information Disclosure - Suspicious Comments** | Info | CWE-615 | Revisado — falso positivo del escáner (comentario de código legítimo del tema) | Confidencialidad — sin impacto real en este caso |
| **Charset Mismatch** (endpoint oEmbed) | Info | CWE-436 | No explotado | Integridad de interpretación de contenido |

### 4. La brecha real: lo que ZAP no vio — y lo que el lado red no pudo confirmar

Este es el valor central de cruzar ambos lados:

- **ZAP no detectó la cadena completa de compromiso.** Marcó Directory Browsing como hallazgo aislado (Medium), pero no tiene forma de saber que esa carpeta expone un `notes.txt`/`user.txt` utilizables como wordlist de fuerza bruta contra SSH — eso requiere razonamiento de negocio y un segundo servicio (SSH) fuera del alcance de un DAST web.
- **ZAP no evaluó SSH en absoluto** — ni la ausencia de MFA, ni el bloqueo de cuenta, ni la fuerza bruta exitosa. Un DAST HTTP no tiene visibilidad sobre otros servicios del mismo host.
- **ZAP no evaluó post-explotación** (enumeración de privilegios, `/opt/L/`, la credencial oculta en `case.wav`, el modelo de sudo de `kira`): son hallazgos de una fase a la que un DAST externo, por definición, no llega.
- **Al revés — el lado red no logró confirmar ni descartar el único hallazgo High del ZAP.** Un primer intento con `sqlmap` contra `comment_post_ID` en `wp-comments-post.php`, a nivel 3/riesgo 2, no reprodujo la inyección. Hay un factor de ruido identificado en esa corrida (protección anti-duplicados de WordPress generando ~1600 respuestas 409, que pudieron opacar señal boolean-based real). Se lanzó un segundo intento más agresivo (nivel 5/riesgo 3, con randomización de contenido y tamper scripts para reducir ese ruido), pero la sesión se cerró antes de obtener su resultado final — es el último análisis que se realizó sobre este hallazgo. Por eso **no puede llamarse falso positivo** con la evidencia disponible: falta la pieza final para cerrar la verificación, y cualquier conclusión en este punto sería prematura. Confirmarlo o descartarlo con confianza requeriría retomar esa corrida o repetir el análisis con un método que aísle mejor el ruido identificado.

**Conclusión:** ZAP cubrió bien la capa de higiene HTTP (cabeceras, directory listing, banners) pero cero cobertura sobre SSH y post-explotación — y su hallazgo más severo quedó, al cierre de este reporte, sin confirmación manual. Esto refuerza el valor de combinar DAST con explotación manual en ambas direcciones: no solo para encontrar lo que el escáner se perdió, sino para auditar la precisión de lo que sí reportó.

---

## 5. Recomendaciones priorizadas

| Prioridad | Recomendación | Hallazgo que mitiga |
|---|---|---|
| 1 | Deshabilitar directory listing (`Options -Indexes`) en `wp-content/uploads/` y subcarpetas | Directory Browsing — vector de acceso inicial completo |
| 2 | Forzar MFA y/o bloqueo de cuenta tras N intentos fallidos en SSH; eliminar credenciales débiles/reutilizadas | Vector de explotación completo (fases 4–7) |
| 3 | Agregar cabeceras de seguridad: `Content-Security-Policy`, `X-Frame-Options`/`frame-ancestors`, `X-Content-Type-Options: nosniff` | 3 hallazgos Medium/Low de ZAP |
| 4 | Agregar tokens anti-CSRF al formulario de comentarios de WordPress | Absence of Anti-CSRF Tokens |
| 5 | Migrar el sitio a HTTPS | HTTP Only Site |
| 6 | Restringir el modelo de sudo de `kira` al mínimo necesario (principio de menor privilegio) en vez de acceso total | Paso a root trivial post-compromiso |
| 7 | No depender de ofuscación (hex/base64 en un archivo de audio) como control de seguridad para credenciales | Hallazgo post-explotación — `case.wav` |
| 8 | Confirmar o descartar formalmente el SQLi de `comment_post_ID`: retomar el análisis agresivo que quedó inconcluso, o repetirlo con un método que aísle mejor el ruido de los 409 | SQL Injection — sin cerrar, requiere análisis más profundo |

---

## 6. Referencias

- Walkthrough técnico completo (lado red): [`sesion-pentestlab-deathnote.md`](./sesion-pentestlab-deathnote.md)
- Reporte DAST original: `ZAP-Scan-Report.pdf` (esta carpeta)
- CWE-89, CWE-548, CWE-352, CWE-693, CWE-1021, CWE-311, CWE-497, CWE-436, CWE-20, CWE-615 — [cwe.mitre.org](https://cwe.mitre.org/)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Cross-Site Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
