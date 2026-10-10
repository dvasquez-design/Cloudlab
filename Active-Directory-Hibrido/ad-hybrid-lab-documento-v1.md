---
title: "Lab Híbrido AD — Documento v1 (Nivel 1 completo)"
tags: [active-directory, windows-server, qemu-kvm, aws, wireguard, cloudformation, ubuntu, realmd, sssd, homelab]
level: N1
status: Nivel 1 completo — corte 2026-10-10 (4 clones unidos, Samba por depto, tráfico, 5 tickets N1 y cierre hechos; DC y túnel apagados)
related: ["[[Lab Híbrido AD — Nivel 1 (Intermedio)]]", "[[Referencia Operativa Maestra]]", "[[Túnel WireGuard — patrón reusable]]", "[[Lab Híbrido AD — Plan final]]"]
absorbe: ["handoff-hybrid-ad-lab.md", "ad-windows-server-bitacora-fases-a-c.md", "handoff-sesion-2026-10-10.md"]
---

# Lab Híbrido AD — Documento v1 (Nivel 1 completo)

> [!abstract] Alcance
> Registro paso a paso de todo lo ejecutado hasta el estado actual: planificación y corrección de conflictos, preparación del disco y el DC, deploy de la infraestructura AWS, túnel WireGuard, EC2 modelo y script de configuración por departamento. Incluye errores, decisiones y comandos exactos. Las secciones 0 a 12 y los Anexos A a D conservan la bitácora cronológica tal como se ejecutó (corte 2026-10-08). **Lo ocurrido el 2026-10-10 (clones, Samba, tráfico, 5 tickets N1, cierre) está en los Anexos E y F**, y el estado vigente en la sección 0.

> [!note] Este documento reemplaza al handoff y a la bitácora Fases A-C
> Las Fases A-C originales están en el Anexo A y lo que traía el handoff (Nivel 2, pendientes, preferencias de trabajo) en el Anexo B. Lo que falta se trabaja paso a paso en el chat (resumen en la sección 12). La guía Nivel 1 original ya no hace falta: lo útil está en el Anexo C.

---

## Bloque para retomar en un chat nuevo

Pegar esto (más este documento, `hybrid-ad-lab-network.yaml` y la Referencia Operativa Maestra) al abrir el chat:

```
Proyecto: Lab Híbrido AD. El Nivel 1 está COMPLETO (corte 2026-10-10). DC win-dc01 (192.168.50.10, KVM, red lab-hybrid/virbr-lab) ↔ túnel wg-hybrid (10.91.0.0/24, host .2, gateway .1) ↔ AWS VPC 10.0.0.0/16, subnet privada 10.0.1.0/24 con 4 clones Ubuntu 24.04 (IT .11, Ventas .12, RRHH .13, Gerencia .14) unidos al dominio corp.wonderland.local en su OU, Samba por departamento (valid users = @"CORP\GG_<dept>"), tráfico simulado y 5 tickets N1 resueltos y documentados (Anexo F).
Usuarios: jperez(IT) msoto(Ventas) pdias(RRHH) arojas(Gerencia) y lgomez (alta del ticket 4, Ventas); grupos GG_<dept>; pass del lab Wonderland#2026!Lab (msoto quedó con Nueva#2026!Lab por el ticket 1).
Estado de infraestructura: DC y túnel apagados; el sandbox de AWS se destruye solo (no se limpia a mano). Si se reconstruye: Anexo E.4 (stack con hybrid-ad-lab-network.yaml, que es la v3 con DeployClients).
Cambios persistentes del host: hook de libvirt reescrito (reinserta los accept al arrancar pentest-lab o lab-hybrid; backup network.bak2) y entrada fstab del USB LABDISK (backup fstab.bak; validar en el próximo arranque). Política de dominio: MinPasswordLength 8, lockout 5. RAM del DC: 6 GB (subir a 9 GB solo con el host libre).
Pendiente: Nivel 2 (Route 53 Resolver / DNS híbrido), VPC endpoints para SSM, monitoreo v2, probar stty -echo del template v3 en la próxima reconstrucción, publicación.
Reglas: solo aditivo, no tocar wg0/wg-azure, no usar nft flush, no reiniciar lab-hybrid con el DC corriendo, PowerShell x64, comandos del DC en líneas cortas, SSH a los clones por el túnel, revisar virsh list --all antes de depurar red. AWS por consola. Monitoreo = v2.
Preferencias: solo el comando sin preámbulo, directo, español, texto plano (sin step_card).
```

---

## 0. Resumen ejecutivo

| Componente | Estado |
|---|---|
| DC `win-dc01` (Windows Server 2022, `corp.wonderland.local`) | ✅ Operativo. 4 OUs, 4 usuarios habilitados, 4 grupos `GG_*`, lockout en 5 |
| Stack CloudFormation (VPC, gateway, SGs; `Active-Directory-Enterprise` el 08-10, recreado como `ActiveDirectoryLab` el 10-10 con la v3) | ✅ Desplegado por consola; el sandbox lo destruye solo |
| Gateway EC2 WireGuard (`hybrid-ad-lab-gateway`) | ✅ Operativo, ENIs `ens5` (WAN) / `ens6` (LAN) |
| Túnel `wg-hybrid` (host ↔ gateway) | ✅ Handshake OK, ping al gateway y al DC desde ambos lados |
| EC2 modelo `ubu-model` (10.0.1.10) | ✅ Paquetes instalados, DC alcanzable (ping, 5 puertos, SRV LDAP) |
| `dept-config.sh` | ✅ v3 con join por Samba y DNS solo-DC; en el template v3 va embebido en el user data de cada clon (9.1, 9.2, E.4) |
| Clon `ubu-it-01` (10.0.1.11) | ✅ Unido a `OU=IT`, `kinit`, `id` y `su` verificados (9.3) |
| Hora del DC | ✅ Zona Chile + NTP; desfase de 4 h corregido (9.4) |
| Clones Ventas / RRHH / Gerencia | ✅ Unidos en su OU; matriz de acceso correcta en los 4 (E.1, F.2) |
| Samba por departamento | ✅ El usuario del depto entra a su share; otro recibe `NT_STATUS_ACCESS_DENIED` (E.3) |
| Tráfico simulado | ✅ Clones y DC; evidencia `ad-lab.pcap` (se conserva local, no va a git) |
| 5 tickets N1 | ✅ Resueltos y documentados (F.1) |
| Cierre (checklist, hook persistente, política 8, RAM, USB) | ✅ F.2 y F.3; DC y túnel apagados |

**Mapa de direcciones final (sin solapes):**

| Qué | Valor |
|---|---|
| DC `win-dc01` | `192.168.50.10` (gw libvirt `.1`, DHCP `.100-.200`, bridge `virbr-lab`) |
| Túnel hybrid (`wg-hybrid`) | gateway `10.91.0.1` / Wonderland `10.91.0.2` |
| VPC / pública / privada | `10.0.0.0/16` / `10.0.0.0/24` / `10.0.1.0/24` |
| Modelo | `10.0.1.10` |
| Clones (IP fija) | IT `.11`, Ventas `.12`, RRHH `.13`, Gerencia `.14` en `10.0.1.0/24` |
| Ya ocupados por otros proyectos | `10.90.0.0/24` (PentestLab), `10.92.0.0/24` (Azure), `10.80.0.0/16` (VNet Azure), `10.10.13.0/24` (pentest-lab) |

---

## 1. Punto de partida y regla de trabajo

Documentación de partida: guía Nivel 1 (`nivel1-hybrid-ad-lab-intermedio.md`), handoff, template CloudFormation base, patrón WireGuard de PentestLab y la Referencia Operativa Maestra.

> [!warning] Regla central del proyecto
> **Aditivo, nunca destructivo.** No tocar `wg0` (PentestLab) ni `wg-azure`, no borrar stacks existentes, no usar `nft flush` (ver §2.4), no reutilizar rangos de otros proyectos. La Referencia Operativa Maestra es la fuente para evitar IPs duplicadas.

Alcance acordado: túnel WireGuard funcional en ambos lados, EC2 modelo replicable con configuración diferencial por departamento, simulación de tráfico, simulación de tickets N1. **Monitoreo queda fuera (v2).**

---

## 2. Conflictos detectados antes de ejecutar (y cómo se resolvieron)

Nada estaba desplegado todavía, así que todas las correcciones fueron previas y no rompieron nada.

### 2.1 Interfaz `wg0` ocupada
- **Problema:** la guía Nivel 1 usa `wg0` + `192.168.99.x`. `wg0` ya es PentestLab (`10.90.0.2`) y `wg-azure` es Azure.
- **Decisión:** este túnel usa la interfaz **`wg-hybrid`** (`/etc/wireguard/wg-hybrid.conf`) con rango `10.91.0.0/24`. El `192.168.99.x` del doc Nivel 1 queda obsoleto.

### 2.2 Hook de libvirt
- **Problema:** el ejemplo del patrón WireGuard usa `10.90.0.0/24` para `lab-hybrid`, incorrecto aquí.
- **Decisión:** agregar un bloque `lab-hybrid` con `10.91.0.0/24` y `10.0.0.0/16`, **sin tocar los bloques existentes**. Antes de editar, leer el hook actual con `sudo cat /etc/libvirt/hooks/network`.

### 2.3 Template CloudFormation: 4 parches antes del primer deploy

| # | Parche | Por qué |
|---|---|---|
| 1 | Peer del gateway: `AllowedIPs = 10.91.0.2/32, 192.168.50.0/24` | Sin el `/32` el gateway no puede responder al ping de `10.91.0.2` |
| 2 | Ruta `0.0.0.0/0` en `PrivateRouteTable` hacia la ENI LAN del gateway | Sin ella los clientes no tienen `apt` ni SSM |
| 3 | ICMP desde `OnPremCidr` y `TunnelCidr` en `PrivateSecurityGroup` | Sin eso falla la verificación por ping |
| 4 | Detección dinámica de interfaces en UserData | El template usaba `eth0/eth1`; en AL2023 son `ens5/ens6` |

### 2.4 Error ya conocido: no usar `nft flush`
La tabla `libvirt_network` es global: un `nft flush chain ip libvirt_network guest_input` borra las reglas de **todas** las redes libvirt. Si algo se rompe, se reinician **`pentest-lab` y `lab-hybrid` juntas**.

### 2.5 Diagrama adjunto descartado como fuente
El diagrama original mezclaba PentestLab con Monitoring (subnets `10.91.0.1/24`, `wg0 10.90.0.2`, una Monitoring instance). No se usó como fuente técnica; se trabajó con la referencia operativa y el handoff. Se rehízo en el entregable de infraestructura (ver diagrama final).

### 2.6 Clientes: CLI vs CloudFormation vs consola
- Plan inicial: `run-instances` por CLI (CFN no tiene `iam:PassRole` en el sandbox; el CLI sí puede pasar `LabInstanceProfile`).
- Decisión posterior del usuario: **las operaciones AWS se hacen por la Management Console.** El stack base queda intacto y el modelo se lanzó por consola.
- Cliente = **Ubuntu** (decisión del handoff, por costo), unido con `realmd/sssd`, no join nativo Windows.

---

## 3. Fase 0 — Disco y DC

### 3.1 Comandos ejecutados

```bash
lsblk -f
sudo mkdir -p /run/media/diego/LABDISK && mountpoint -q /run/media/diego/LABDISK || sudo mount LABEL=LABDISK /run/media/diego/LABDISK
sudo chmod o+x /run/media /run/media/diego /run/media/diego/LABDISK
ls -la /run/media/diego/LABDISK/ && sudo ufw status verbose | grep -E "virbr-lab|Default"
sudo virsh net-list --all && sudo virsh list --all
```

### 3.2 Resultado

- `sdd1` ext4, label `LABDISK`, ya montado en `/run/media/diego/LABDISK` (53 GB libres, 58 % usado). `sdb1`/`sdc1` son exFAT (otros discos; el de labs es el ext4).
- En el disco: `win-dc01.qcow2` (60 GB, propietario `libvirt-qemu`), directorio `parrot`, `lost+found`.
- UFW: `Default: deny (incoming), allow (outgoing), allow (routed)`. Reglas `ALLOW FWD` presentes entre `wlp0s20u2` y `virbr-lab` (IPv4 e IPv6).
- Redes libvirt: `default` inactiva, `lab-hybrid` activa + autostart, `pentest-lab` activa.
- VMs: todas `shut off` (`deathnote`, `metasploitable2`, `mrrobot`, `ubuntu24.04`, `win-dc01`, `win10`).

> [!note] Recordatorio de causa raíz #2 (handoff)
> Los permisos del USB se pierden en cada remount **en toda la cadena** (`/run/media`, `/run/media/diego`, `LABDISK`). Por eso los tres `chmod o+x`. Si libvirt sigue diciendo `Permission denied (as uid:950, gid:950)`, agregar `sudo chown 950:950 /run/media/diego/LABDISK/win-dc01.qcow2` (en esta sesión el `ls -la` mostró el qcow2 ya con propietario `libvirt-qemu`, así que no hizo falta). Pendiente histórico: regla `udev` para automatizarlo.

### 3.3 Veredicto
Fase 0 OK. `win-dc01` se arrancó después con `sudo virsh start win-dc01` (consola vía virt-manager, **PowerShell de 64 bits**, sin "(x86)" en el título).

---

## 4. Fase 1 — Preparación del template y claves

### 4.1 Pregunta clave: ¿por qué un template nuevo (v2)?
Se generó `hybrid-ad-lab-network-v2.yaml` en lugar de reutilizar el base porque llevaba los 4 parches de §2.3. El usuario cuestionó el diseño de claves y aportó `pentest-template.yaml`.

### 4.2 Hallazgo: la referencia operativa estaba desactualizada en un punto
`pentest-template.yaml` **genera la clave privada del server en el primer boot** y solo recibe la pública del cliente como parámetro. La Referencia Operativa Maestra decía que ambas claves iban hardcodeadas como default (`WgServerPrivateKey`/`WgClientPublicKey`). Se corrigió la referencia (ver su sección 2).

### 4.3 Cambio en v2 por ese hallazgo
- **Quitado:** parámetro `WgServerPrivateKey`. Ninguna clave privada pasa por CloudFormation.
- **Agregado:** el UserData genera el par con `wg genkey` en `/etc/wireguard/server_private.key` y escribe `wg0.conf` leyéndolo desde el archivo.
- **Agregado:** Output `WgServerPublicKeyNote` (la pública del server nace en el primer boot; se obtiene por SSM con `sudo cat /etc/wireguard/server_public.key`).

### 4.4 Parámetros relevantes del template final

| Parámetro | Valor |
|---|---|
| `VpcCidr` / `PublicSubnetCidr` / `PrivateSubnetCidr` | `10.0.0.0/16` / `10.0.0.0/24` / `10.0.1.0/24` |
| `OnPremCidr` | `192.168.50.0/24` |
| `TunnelCidr` | `10.91.0.0/24` (distinto de `10.90.0.0/24` PentestLab y `10.92.0.0/24` Azure) |
| `WgServerAddress` / `WgClientTunnelIp` | `10.91.0.1/24` / `10.91.0.2/32` |
| `WireGuardPort` | `51820/udp` |
| `KeyName` | `AD` (key pair, sin `.pem`) |
| `GatewayInstanceType` | `t3.micro` (AMI AL2023 vía parámetro SSM) |
| `ExistingInstanceProfileName` | vacío a propósito (CFN sin `iam:PassRole` en el sandbox) |
| `WgClientPublicKey` | sin default, se pasa siempre explícita |

Recursos: VPC, IGW, subnet pública/privada, route tables (pública con IGW; privada con ruta a `192.168.50.0/24` y `0.0.0.0/0` hacia la ENI LAN), `GatewayWanSecurityGroup` (solo `UDP/51820` desde `0.0.0.0/0`, por CGNAT), `PrivateSecurityGroup` (DNS 53, Kerberos 88, LDAP 389, SMB 445, kpasswd 464 desde on-prem; SSH 22 e ICMP desde el túnel; ICMP desde on-prem; self-ingress), gateway con **dos ENIs** (`SourceDestCheck: false`), EIP asociada a la ENI WAN.

### 4.5 Claves del cliente
Antes de generar nada se verificó si ya existían claves (`ls /etc/wireguard`). Se usan archivos propios `hybrid_*`, **sin tocar** las de `wg0` ni `wg-azure`:

```bash
wg genkey | sudo tee /etc/wireguard/hybrid_client_private.key >/dev/null && sudo cat /etc/wireguard/hybrid_client_private.key | wg pubkey | sudo tee /etc/wireguard/hybrid_client_public.key
sudo cat /etc/wireguard/hybrid_client_public.key
```

> [!tip] Regla de oro del patrón WireGuard
> Nunca transcribir claves a mano. Generar → `cat` para verificar → que todo lo siguiente lea el archivo (`$(sudo cat ...)`).

---

## 5. Fase 0.5 — Dejar el AD listo antes del deploy

El usuario pidió **pausar el deploy** y cerrar primero el Active Directory. Las operaciones AWS se harían por consola.

### 5.1 Revisión de solo lectura en el DC (PowerShell x64)

```powershell
Get-ADDomain | Select Name,DomainMode,PDCEmulator; dcdiag /q; Get-ADOrganizationalUnit -Filter * | Select Name; Get-ADUser -Filter * | Where DistinguishedName -like "*OU=*" | Select SamAccountName,Enabled,DistinguishedName; Get-ADDefaultDomainPasswordPolicy | Select MinPasswordLength,ComplexityEnabled,LockoutThreshold,LockoutDuration; Get-GPO -All | Select DisplayName; Get-DnsServerForwarder; Get-NetIPConfiguration; w32tm /query /status
```

### 5.2 Primer intento de cambios aditivos (con errores)

```powershell
$d="DC=corp,DC=wonderland,DC=local"; foreach ($g in "IT","Ventas","RRHH","Gerencia") { New-ADGroup -Name "GG_$g" -GroupScope Global -GroupCategory Security -Path "OU=$g,$d" }; Add-ADGroupMember GG_IT jperez; Add-ADGroupMember GG_Ventas msoto; Add-ADGroupMember GG_RRHH pdiaz; Add-ADGroupMember GG_Gerencia arojas; "jperez","msoto","pdiaz","arojas" | ForEach-Object { Set-ADUser $_ -ChangePasswordAtLogon $false }; New-ADReplicationSubnet -Name "10.0.1.0/24" -Site "Default-First-Site-Name"; Enable-NetFirewallRule -Name FPS-ICMP4-ERQ-In
```

### 5.3 Errores y hallazgos (con capturas)

| # | Hallazgo | Causa | Resolución |
|---|---|---|---|
| 1 | Los 4 usuarios estaban `Enabled = False` | Causa no confirmada. La bitácora A-C dice que `Password123!` cumple complejidad y se creó con `-Enabled $true`; la hipótesis de la política de contraseñas es probable pero no verificada | Reset de contraseña + `Enable-ADAccount` |
| 2 | `pdiaz` no existía | El CSV creó **`pdias`** ("Pedro Dias"), no `pdiaz` | **De aquí en adelante todo el lab usa `pdias`** |
| 3 | `LockoutThreshold = 0` | Sin bloqueo de cuentas por defecto | Se fijó a 5 (necesario para el ticket 2) |
| 4 | Los grupos `GG_*` no se crearon | El `-Path` evaluado fue `"*OU=*"` (filtro mal interpretado) | Nuevo comando con path explícito por OU |
| 5 | `dcdiag /q`: 3 eventos de arranque (16:39, ADWS e IKEEXT) | ADWS aún no estaba arriba al arrancar; ya respondía (`Get-ADUser` funcionaba) | Esperable. Único test fallido: `SystemLog`, esperable |
| 6 | Subnet `10.0.1.0/24` e ICMP | Ya aplicados por el primer intento | Nada que hacer |

### 5.4 Comandos correctos aplicados

Contraseña común del lab: `Wonderland#2026!Lab` (20 caracteres, pasa cualquier mínimo).

```powershell
$p = ConvertTo-SecureString "Wonderland#2026!Lab" -AsPlainText -Force
Get-ADUser -Filter * | ? DistinguishedName -like "*OU=*" | % { Set-ADAccountPassword $_ -Reset -NewPassword $p; Set-ADUser $_ -ChangePasswordAtLogon $false; Enable-ADAccount $_ }
"IT","Ventas","RRHH","Gerencia" | % { New-ADGroup -Name "GG_$_" -GroupScope Global -Path "OU=$_,DC=corp,DC=wonderland,DC=local" }
"IT","Ventas","RRHH","Gerencia" | % { $o=$_; Get-ADUser -Filter * -SearchBase "OU=$o,DC=corp,DC=wonderland,DC=local" | % { Add-ADGroupMember "GG_$o" $_ } }
Set-ADDefaultDomainPasswordPolicy -Identity corp.wonderland.local -LockoutThreshold 5 -LockoutDuration 00:10:00 -LockoutObservationWindow 00:10:00
```

> [!note] Por qué `ChangePasswordAtLogon $false`
> Los usuarios se habían creado con contraseña vencida; `kinit` y `sssd` fallarían con "password expired" en la simulación de tráfico.

### 5.5 Verificación (3 capturas)

```powershell
Get-ADUser -Filter * -Properties PasswordExpired | ? DistinguishedName -like "*OU=*" | select SamAccountName,Enabled,PasswordExpired
"IT","Ventas","RRHH","Gerencia" | % { "GG_$_"; Get-ADGroupMember "GG_$_" | select -Expand SamAccountName }
Get-ADDefaultDomainPasswordPolicy | select MinPasswordLength,LockoutThreshold
```

**Resultado:** 4 usuarios habilitados y sin contraseña vencida; `GG_IT`=`jperez`, `GG_Ventas`=`msoto`, `GG_RRHH`=`pdias`, `GG_Gerencia`=`arojas`; lockout en 5.

### 5.6 Estado final del AD

| Elemento | Valor |
|---|---|
| Dominio / NetBIOS | `corp.wonderland.local` / `CORP` |
| Modo de dominio | `Windows2016Domain` |
| OUs | IT, Ventas, RRHH, Gerencia |
| Usuarios | `jperez` (IT), `msoto` (Ventas), `pdias` (RRHH), `arojas` (Gerencia) |
| Grupos | `GG_IT`, `GG_Ventas`, `GG_RRHH`, `GG_Gerencia` |
| Política | Lockout 5 intentos / 10 min duración / 10 min ventana |
| Sitio AD | Subnet `10.0.1.0/24` en `Default-First-Site-Name` |
| Firewall DC | `FPS-ICMP4-ERQ-In` habilitada |

---

## 6. Fase 1 — Deploy del stack (por consola)

### 6.1 Verificación de credenciales y stack existente
Se advirtió: el stack se llama **`Active-Directory-Enterprise`** (nombre de la memoria del proyecto). Si ya existía, **no borrarlo**: el stack `monitoring-AD` importa su VPC. El deploy resultó exitoso.

### 6.2 Pasos en la Management Console

1. CloudFormation → Create stack → *With new resources (standard)*.
2. *Upload a template file* → `hybrid-ad-lab-network-v2.yaml`.
3. Stack name `Active-Directory-Enterprise`. Único parámetro a completar: `WgClientPublicKey` (contenido de `hybrid_client_public.key`).
4. Next → Next → Submit (sin recursos IAM, no pide capabilities).
5. Esperar `CREATE_COMPLETE` (3-5 min). Anotar de Outputs: `GatewayWanPublicIp`, `PrivateSecurityGroupId`, `GatewayInstanceId`.
6. EC2 → `hybrid-ad-lab-gateway` → Actions → Security → **Modify IAM role** → `LabInstanceProfile`.
7. Reboot de la instancia; esperar status checks 2/2.

> [!warning] Checklist manual del sandbox
> `LabInstanceProfile` **no** puede asignarse desde CloudFormation (sin `iam:PassRole`). Siempre es manual + reboot.

### 6.3 IDs resultantes

| Recurso | ID |
|---|---|
| VPC | `vpc-04b40726255364691` |
| Subnet privada | `subnet-01db8cd3c386bd235` |
| `PrivateSecurityGroup` | `sg-0aac46ef60ebc434a` |
| Gateway (hostname interno) | `ip-10-0-0-104` |

---

## 7. Fase 2 — WireGuard y reglas del host

### 7.1 Reglas UFW aditivas (la interfaz `wg-hybrid` aún no existía; UFW las acepta igual)

```bash
sudo ufw route allow in on wg-hybrid out on virbr-lab
sudo ufw route allow in on virbr-lab out on wg-hybrid
sudo cat /etc/libvirt/hooks/network
```

El `cat` del hook se hizo **antes de editarlo** para agregar solo el bloque `lab-hybrid` (aceptar `10.91.0.0/24` y `10.0.0.0/16` hacia `virbr-lab`, con `oifname`, no `oif`).

> [!warning] A confirmar
> El texto pegado no muestra el contenido final del hook ni de `wg-hybrid.conf`. Dado que el tráfico host↔DC por el túnel funciona, se asume aplicados; **verificar con `sudo cat /etc/libvirt/hooks/network` y `sudo cat /etc/wireguard/wg-hybrid.conf`** y registrarlos en la referencia operativa.

### 7.2 Configuración del lado cliente (plan acordado)

`/etc/wireguard/wg-hybrid.conf` (escrito con heredoc controlado, sin transcribir claves):
- `Address = 10.91.0.2/24`
- `Peer`: pública del server (`/etc/wireguard/server_public.key` del gateway, obtenida por SSM), `Endpoint = <GatewayWanPublicIp>:51820`
- `AllowedIPs = 10.91.0.0/24, 10.0.0.0/16`, `PersistentKeepalive = 25`

### 7.3 Estado del túnel
- Levantado a mano con `sudo wg-quick up wg-hybrid`. **No se habilitó el servicio permanente** para no dejar `wg-hybrid` arrancando solo junto a `wg0` y `wg-azure`.
- Handshake hace 9 s; `ping 10.91.0.1` responde. Esto solo probaba host ↔ gateway.

### 7.4 Troubleshooting host → DC

**Síntoma:** desde el host `ping -c 2 192.168.50.10` → 100 % pérdida. Desde el gateway (SSM):

```
WAN_IF=ens5 LAN_IF=ens6
-P FORWARD ACCEPT
-A FORWARD -i ens6 -o wg0 -j ACCEPT
-A FORWARD -i wg0 -o ens6 -j ACCEPT
-A FORWARD -i ens6 -o ens5 -j ACCEPT
-A FORWARD -i ens5 -o ens6 -m state --state RELATED,ESTABLISHED -j ACCEPT
-P POSTROUTING ACCEPT
-A POSTROUTING -o ens5 -j MASQUERADE
ping 192.168.50.10 → From 10.91.0.2 Destination Host Unreachable
```

**Diagnóstico:** el gateway estaba correcto (detectó `ens5`/`ens6`, FORWARD y MASQUERADE bien) y el paquete llegó al host por el túnel. El `Destination Host Unreachable` venía de `10.91.0.2` (el propio host): el último tramo host → DC fallaba, y el ping local al DC tampoco respondía, es decir **la VM no contestaba en `virbr-lab`**.

**Comandos de diagnóstico:**
```bash
sudo virsh list --all
ip neigh show dev virbr-lab
sudo virsh domiflist win-dc01
```
```powershell
ipconfig
```

**Causa raíz:** la VM `win-dc01` estaba **pausada** (el usuario la había pausado). Al reanudarla, ambos pings (host y gateway SSM) funcionaron.

> [!tip] Lección (se suma a la lista de causas raíz)
> Antes de depurar firewall/ruteo/túnel, `sudo virsh list --all` — una VM `paused` produce exactamente este síntoma (ARP sin respuesta).

### 7.5 Nota sobre `-P FORWARD ACCEPT` en el gateway
La política por defecto de FORWARD es `ACCEPT`; las reglas explícitas documentan la intención, pero el filtrado real lo hace el Security Group. Si se endurece el gateway en el futuro, cambiar a `DROP` y mantener solo las reglas explícitas.

### 7.6 Camino verificado
Host y gateway llegan al DC por el túnel. **El túnel dejó de ser el riesgo.**

---

## 8. Fase 3 — EC2 modelo

### 8.1 Configuración de lanzamiento (consola)

| Campo | Valor |
|---|---|
| Name | `ubu-model` |
| AMI | Ubuntu Server 24.04 LTS x86_64 |
| Tipo | `t3.micro` |
| Key pair | `AD` |
| VPC / Subnet | `vpc-04b40726255364691` / `subnet-01db8cd3c386bd235` (privada) |
| IP pública | Disable |
| Security group | existente `sg-0aac46ef60ebc434a` |
| Primary IP | `10.0.1.10` |
| IAM instance profile | `LabInstanceProfile` |
| Credit specification | **Unlimited** (Standard recomendado; el usuario usa Unlimited en el sandbox para evitar problemas de crédito) |
| Allow tags in metadata | Enable (para que cada clon lea su tag `Dept`) |

### 8.2 User data del modelo (sin unir al dominio)

```bash
#!/bin/bash
exec > /var/log/model-setup.log 2>&1
set -x
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y realmd sssd sssd-tools sssd-ad adcli krb5-user samba samba-common-bin oddjob oddjob-mkhomedir packagekit libnss-sss libpam-sss chrony dnsutils ldap-utils smbclient cifs-utils netcat-openbsd
systemctl disable --now smbd nmbd
echo ready > /var/lib/model-ready
```

> [!warning] El modelo NO se une al dominio
> Unir el modelo y clonarlo genera colisión de keytab (`/etc/krb5.keytab`) y de cuenta de equipo. Cada clon se une por su cuenta con `dept-config.sh`.

### 8.3 Verificación (instancia `running` 3/3, SSM)

```bash
cat /var/lib/model-ready; tail -3 /var/log/model-setup.log
ping -c 2 192.168.50.10; nc -zv 192.168.50.10 53 88 389 445 464
dig +short @192.168.50.10 -t SRV _ldap._tcp.corp.wonderland.local
```
**Resultado:** ping al DC OK, los 5 puertos abiertos (53, 88, 389, 445, 464), SRV de `_ldap._tcp` resuelve a `win-dc01`. Desde la subnet privada se alcanzan DNS, Kerberos y LDAP del DC a través del túnel.

---

## 9. `dept-config.sh` — configuración diferencial por departamento

Instalado en el modelo (`/usr/local/bin/dept-config.sh`). Uso: `sudo dept-config.sh <IT|Ventas|RRHH|Gerencia>`; sin argumento lee el tag `Dept` de IMDS.

```bash
sudo tee /usr/local/bin/dept-config.sh > /dev/null << 'SCRIPT_EOF'
#!/bin/bash
# dept-config.sh - Configura un clon por departamento y lo une a corp.wonderland.local
# Uso: sudo dept-config.sh <IT|Ventas|RRHH|Gerencia>   (sin argumento lee el tag Dept de IMDS)
set -euo pipefail

DOMAIN="corp.wonderland.local"
BASE_DN="DC=corp,DC=wonderland,DC=local"
DC_IP="192.168.50.10"
VPC_DNS="10.0.0.2"
JOIN_USER="${JOIN_USER:-Administrator}"

[ "$(id -u)" -eq 0 ] || { echo "Ejecutar con sudo"; exit 1; }

DEPT="${1:-}"
if [ -z "$DEPT" ]; then
  TOKEN=$(curl -s -m 5 -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60" || true)
  DEPT=$(curl -s -m 5 -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/tags/instance/Dept || true)
fi
case "$DEPT" in
  IT|Ventas|RRHH|Gerencia) ;;
  *) echo "Uso: sudo dept-config.sh <IT|Ventas|RRHH|Gerencia>"; exit 1 ;;
esac
DL="${DEPT,,}"
SHORT="ubu-${DL}-01"
FQDN="${SHORT}.${DOMAIN}"

echo "[1/6] Hostname: $FQDN"
hostnamectl set-hostname "$FQDN"
echo "preserve_hostname: true" > /etc/cloud/cloud.cfg.d/99-preserve-hostname.cfg
IP=$(hostname -I | awk '{print $1}')
sed -i "/ ${SHORT}\$/d" /etc/hosts
echo "$IP $FQDN $SHORT" >> /etc/hosts

echo "[2/6] DNS: $DC_IP primero, $VPC_DNS de respaldo"
IFACE=$(ip -o -4 route show to default | awk '{print $5}' | head -1)
cat > /etc/netplan/60-ad-dns.yaml << NETEOF
network:
  version: 2
  ethernets:
    $IFACE:
      dhcp4-overrides:
        use-dns: false
      nameservers:
        addresses: [$DC_IP, $VPC_DNS]
        search: [$DOMAIN]
NETEOF
chmod 600 /etc/netplan/60-ad-dns.yaml
netplan generate || { rm -f /etc/netplan/60-ad-dns.yaml; echo "netplan invalido, archivo removido"; exit 1; }
resolvectl dns "$IFACE" "$DC_IP" "$VPC_DNS"
resolvectl domain "$IFACE" "$DOMAIN"

echo "[3/6] Descubrimiento del DC"
getent hosts "win-dc01.$DOMAIN" > /dev/null || { echo "No resuelve el DC por DNS (tunel caido?)"; exit 1; }
realm discover "$DOMAIN" | head -3

echo "[4/6] Join al dominio en OU=$DEPT"
if realm list 2>/dev/null | grep -q "domain-name: $DOMAIN"; then
  echo "Ya unido, se omite el join"
else
  PW="${AD_PASS:-}"
  if [ -z "$PW" ]; then read -rsp "Password de $JOIN_USER@$DOMAIN: " PW; echo; fi
  printf '%s' "$PW" | realm join -U "$JOIN_USER" --computer-ou="OU=$DEPT,$BASE_DN" "$DOMAIN"
  unset PW
fi

echo "[5/6] Acceso: solo GG_$DEPT y GG_IT"
pam-auth-update --enable mkhomedir
realm deny --all
ALLOW="GG_${DEPT}@${DOMAIN}"
if [ "$DEPT" != "IT" ]; then ALLOW="$ALLOW GG_IT@${DOMAIN}"; fi
# shellcheck disable=SC2086
realm permit -g $ALLOW

GID_IT=$(getent group "gg_it@${DOMAIN}" | cut -d: -f3 || true)
if [ -n "$GID_IT" ]; then
  echo "%#${GID_IT} ALL=(ALL) ALL" > /etc/sudoers.d/gg-it
  chmod 440 /etc/sudoers.d/gg-it
  visudo -cf /etc/sudoers.d/gg-it
else
  echo "AVISO: no resolvi GG_IT, sudo para IT no configurado"
fi

echo "[6/6] Resumen"
realm list | head -12
getent group "gg_${DL}@${DOMAIN}" || echo "AVISO: no resolvi GG_$DEPT"
SCRIPT_EOF
sudo chmod +x /usr/local/bin/dept-config.sh && bash -n /usr/local/bin/dept-config.sh && echo OK
```

### Qué hace en un clon

| Paso | Acción |
|---|---|
| 1 | Hostname `ubu-<dept>-01` (+ `preserve_hostname`, entrada en `/etc/hosts`) |
| 2 | DNS por netplan (`60-ad-dns.yaml`): DC primero, resolver de la VPC `10.0.0.2` de respaldo. Si `netplan generate` falla, se borra el archivo |
| 3 | Valida resolución del DC antes de intentar el join (si falla, probablemente túnel caído) |
| 4 | `realm join --computer-ou="OU=<dept>,..."`. Idempotente. Password: la pide o toma `AD_PASS` |
| 5 | `realm deny --all` + `realm permit -g GG_<dept> [GG_IT]`. Sudo solo para `GG_IT` vía `/etc/sudoers.d/gg-it` (por GID) |
| 6 | Resumen: `realm list` y resolución del grupo del departamento |

### 9.1 Cambios posteriores al script (aplicados)

1. **Join por membresía Samba**, para poder compartir carpetas con autenticación AD más adelante (aplicado antes de crear la AMI; copia en `/root/dept-config.sh.bak`):
```bash
sudo sed -i 's/realm join -U/realm join --membership-software=samba -U/' /usr/local/bin/dept-config.sh
```
2. **DC como único DNS del clon** (ver 9.3, el DNS de la VPC rompía la resolución del dominio):
```bash
sudo sed -i 's/addresses: \[\$DC_IP, \$VPC_DNS\]/addresses: [$DC_IP]/; s/"\$DC_IP" "\$VPC_DNS"$/"$DC_IP"/' /usr/local/bin/dept-config.sh
```
Efecto colateral aceptado: si cae el túnel, el clon pierde también la resolución de internet (útil para el ticket 3). La línea de mensaje `[2/6]` todavía menciona el respaldo; es solo texto.

### 9.2 AMI bloqueada por el sandbox → plan B

- La AMI `ubu-ad-model` se creó y llegó a `available`.
- Al lanzar un clon desde ella: `You are not authorized to perform this operation ... ec2:RunInstances on resource arn:aws:ec2:us-west-2::image/ami-0dd4a0e4347e94af1 because no identity-based policy allows the ec2:RunInstances action.` El sandbox solo deja lanzar desde AMIs permitidas (la de Ubuntu sí).
- **Plan B:** cada clon se lanza desde Ubuntu Server 24.04 con un user data que instala los paquetes y escribe `dept-config.sh` (con la variante Samba, sin el ajuste de DNS). Marca de fin: `/var/lib/clone-ready`; el script queda en `/usr/local/bin/dept-config.sh` (2768 bytes).
- En el asistente de lanzamiento los tags están en "Name and tags" → **Add additional tags** (no existe una sección "Resource tags" aparte).
- Los clones siguientes se lanzan con **Launch more like this** desde `ubu-it-01` (copia AMI, subnet, SG, perfil y user data); se cambian Name, Primary IP y el tag `Dept`.
- Limpieza al cierre: deregistrar `ubu-ad-model` y borrar su snapshot.

### 9.3 Join de `ubu-it-01` (con error de DNS)

**Primer intento:** `[3/6] No resuelve el DC por DNS (tunel caido?)`.
- Diagnóstico: `ping 192.168.50.10` OK (≈186 ms) y `dig +short @192.168.50.10 win-dc01.corp.wonderland.local` → `192.168.50.10`, es decir, túnel y DC bien.
- `resolvectl status`: `Current DNS Server: 10.0.0.2` (servidor activo = DNS de la VPC), con `DNS Servers: 192.168.50.10 10.0.0.2`. `resolvectl query win-dc01...` → `Name not found`.
- Prueba: `sudo resolvectl dns ens5 192.168.50.10 && getent hosts win-dc01.corp.wonderland.local` → resolvió. **Causa raíz:** `systemd-resolved` usa el servidor de la VPC, que no conoce `corp.wonderland.local`. (El DNS híbrido nativo con Route 53 Resolver es del Nivel 2.)
- Fix: cambio 2 de 9.1 y repetir.

**Segundo intento:** join correcto. Observaciones:
- Aparece un segundo `Password for Administrator:` y una línea de guiones: es el `net ads join` de la membresía Samba; terminó bien.
- `[5/6] Acceso: solo GG_IT y GG_IT` es solo texto repetido para IT.
- `sudo realm list`: `configured: kerberos-member`, `client-software: sssd`, `required-package: samba-common-bin`, `permitted-groups: GG_IT@corp.wonderland.local`.
- `id jperez@corp.wonderland.local` → uid `361401103`, grupo `gg_it@corp.wonderland.local` (gid `361401107`).
- En el DC, `Get-ADComputer`: `UBU-IT-01` en `OU=IT`.
- `kinit jperez@CORP.WONDERLAND.LOCAL` y `klist` OK; `sudo su - jperez@corp.wonderland.local -c 'id; pwd'` crea `/home/jperez@corp.wonderland.local`.

**Hallazgo Samba:** `sudo net ads testjoin` → `Failed to get machine credentials ... Access Denied`. No es un join fallido: `secrets.tdb` y `krb5.keytab` se crearon en el momento del join, pero `/etc/samba/smb.conf` sigue con la configuración por defecto de Ubuntu (`workgroup = WORKGROUP`, `server role = standalone server`), porque `realm` no la reescribe con cliente sssd. Se resuelve en el paso de Samba escribiendo el `smb.conf` a mano (`workgroup = CORP`, `security = ads`, `realm = CORP.WONDERLAND.LOCAL`, `kerberos method`, mapeo de ids para sssd).

### 9.4 Desfase horario del DC (4 horas)

- Síntoma: `klist` mostraba el ticket de `10/09/26 03:47` cuando el clon marcaba `23:4x UTC` del 8. `kinit` funcionaba porque Kerberos se ajusta solo al reloj del KDC, pero era un desfase real.
- Causa: `win-dc01` tenía la zona `Pacific Standard Time` (UTC-7) con el reloj en hora de Chile (20:49), así que calculaba UTC = 03:49. `w32tm` usaba `Local CMOS Clock` (estrato 1, sin NTP). El clon estaba bien (`chronyc`: offset de microsegundos contra el servicio de hora de AWS).
- Fix, en el DC (PowerShell x64):
```powershell
Set-TimeZone -Id "Pacific SA Standard Time"
w32tm /config /manualpeerlist:"0.pool.ntp.org,0x8" /syncfromflags:manual /reliable:yes /update; w32tm /resync /force
```
- Resultado: `20:53:30` local / `23:53:30` UTC, fuente `0.pool.ntp.org`, estrato 3. Los objetos de AD creados hoy quedan con marca de tiempo unas horas en el futuro; no afecta en un DC único.
- Pendiente de verificar: `sudo virsh dumpxml win-dc01 | grep -A1 "<clock"` debería mostrar `offset='localtime'`.

### 9.5 Estado al corte

- `ubu-it-01` completo y verificado.
- `ubu-ventas-01` (10.0.1.12), `ubu-rrhh-01` (10.0.1.13), `ubu-gerencia-01` (10.0.1.14): lanzados con *Launch more like this*; falta confirmar que estén en `running` 3/3 con el tag `Dept` correcto.
- Por clon, en este orden: (1) aplicar el cambio 2 de 9.1; (2) `sudo dept-config.sh <dept>`; (3) `realm list`, `id`, `kinit` y `Get-ADComputer`; (4) matriz de acceso.
- Después: Samba (`smb.conf` manual), tráfico simulado, 5 tickets N1, cierre.

---

## 10. Registro de errores y causas raíz (acumulado)

> [!note] Las filas 18 a 30 corresponden a la sesión del 2026-10-10 (Anexo E). El error 21 (reglas `reject` de libvirt) quedó resuelto de forma persistente en F.3.

| # | Error | Causa raíz | Prevención |
|---|---|---|---|
| 1 | Driver NetKVM no se instala solo | La búsqueda automática pide internet | Browse manual a virtio-win `NetKVM\2k22\amd64` |
| 2 | Permisos del USB se pierden en el remount | Toda la cadena de directorios pierde `x` | 3 `chmod o+x`; pendiente regla `udev` |
| 3 | DNS sin resolver con IP OK | UFW `deny (routed)` sin regla para `virbr-lab` | `ufw route allow` por cada red libvirt nueva |
| 4 | Cmdlets AD "no reconocidos" | Consola PowerShell (x86) | Siempre x64; alternativa `dism` |
| 5 | Usuarios deshabilitados / contraseña vencida | CSV con contraseña que no pasó la política + `ChangePasswordAtLogon` | Reset + `Enable-ADAccount` + `ChangePasswordAtLogon $false` |
| 6 | `pdiaz` inexistente | El CSV tenía `Dias` → `pdias` | Verificar siempre con `Get-ADUser` antes de asumir un SamAccountName |
| 7 | `New-ADGroup` fallido | `-Path` mal interpretado (`"*OU=*"`) | Path explícito por OU en un loop |
| 8 | `LockoutThreshold = 0` | Valor por defecto | `Set-ADDefaultDomainPasswordPolicy` (5 / 10 min) |
| 9 | Host y gateway no llegaban al DC (`Destination Host Unreachable`) | VM `win-dc01` **pausada** | `virsh list --all` antes de depurar la red |
| 10 | Referencia operativa desactualizada (claves WG de PentestLab) | La plantilla real genera la privada del server en el primer boot | Corregida la referencia |
| 11 | `nft flush` borraría todas las redes libvirt | Tabla `libvirt_network` global | No usarlo; reiniciar `pentest-lab` y `lab-hybrid` juntas |
| 12 | `eth0/eth1` en iptables del gateway | AL2023 usa `ens5/ens6` | Detección dinámica en UserData |
| 13 | `ec2:RunInstances` denegado sobre `ami-0dd4a0e4347e94af1` | El sandbox no permite lanzar desde AMIs propias | Lanzar desde Ubuntu 24.04 con user data (9.2) |
| 14 | `dept-config.sh`: "No resuelve el DC por DNS" con túnel y DC sanos | `systemd-resolved` usaba `10.0.0.2` (VPC), que no conoce el dominio | DC como único DNS del clon (9.1) |
| 15 | `net ads testjoin`: "Failed to get machine credentials" | `smb.conf` por defecto (standalone, `WORKGROUP`); `realm` no lo configura con sssd | Configurar `smb.conf` en el paso de Samba (9.3) |
| 16 | Tickets Kerberos con hora 4 h adelantada | DC en zona `Pacific Standard Time` con reloj en hora de Chile y sin NTP | `Set-TimeZone` a Chile + NTP (9.4) |
| 17 | `MinPasswordLength` vigente = 7, no 8 | La GPO `Politica-Contrasenas-Corp` no es la que manda (Default Domain Policy) | **Resuelto el 10-10:** `Set-ADDefaultDomainPasswordPolicy -MinPasswordLength 8` (F.3) |
| 18 | `ping 10.91.0.1` sin respuesta, `0 B received` en `wg show` | El stack no existía: el sandbox se había reiniciado. No tenía que ver con la VM del DC | Redeploy completo. Antes de depurar el túnel, confirmar en la consola que el gateway existe y está `running` |
| 19 | Gateway sin `wg` ni `iptables`; claves del server vacías; `Failed to enable unit: wg-quick@wg0` | En el primer arranque `dnf` no resolvía hostnames (aún sin salida a internet); el user data siguió igual | Template v3: el gateway ahora `DependsOn` de la ruta al IGW, la asociación de la tabla de rutas y la EIP; `dnf install` con hasta 20 reintentos y aborto con mensaje claro si falta `wg` |
| 20 | Change set `FAILED: No updates are to be performed` | Se creó un change set con la misma plantilla y `DeployClients` seguía en `false` | **Make a direct update** → *Use existing template* → `DeployClients=true` |
| 21 | Desde un clon: `ping 192.168.50.10` → `Destination Port Unreachable` desde `10.91.0.2`; DNS `connection refused`. El host sí llega al DC | En `libvirt_network guest_input` el `reject` de `virbr-lab` estaba **antes** de los `accept ip saddr 10.0.0.0/16` y `10.91.0.0/24` (177 paquetes rechazados) | `nft insert rule ... guest_input oifname "virbr-lab" ip saddr 10.0.0.0/16 accept` (y `10.91.0.0/24`). **No** reiniciar `lab-hybrid` con el DC corriendo (le desconecta la interfaz). Es solo en memoria: se pierde al reiniciar el host o las redes |
| 22 | Segundo prompt de contraseña (el de `realm`/Samba) se ve al teclear | `realm join --membership-software=samba` pide la contraseña por su cuenta, leyendo del terminal | Template v3: `stty -F /dev/tty -echo` antes del join y `echo` después, con `trap` de restauración. **Sin probar** |
| 23 | `wbinfo -t`: `trust secret for domain WORKGROUP ... NT_STATUS_NO_SUCH_DOMAIN` (pero `net ads testjoin` = `Join is OK`) | `apt` arrancó `winbind` con el `smb.conf` por defecto; `enable --now` no reinicia un servicio que ya corre | `systemctl restart winbind smbd` después de escribir el `smb.conf` |
| 24 | `smbclient` → `NT_STATUS_LOGON_FAILURE` aunque `wbinfo -a` autenticaba. `log.smbd`: `getpwuid(11103) failed, is nsswitch configured?` | `winbind` mapea al usuario a un UID (idmap rid) que `smbd` no puede resolver: `winbind` no estaba en `nsswitch.conf` | `apt install libnss-winbind` y añadir `winbind` a las líneas `passwd:` y `group:` de `/etc/nsswitch.conf` |
| 25 | `smbclient -L localhost -k` → `ACCESS_DENIED` | Mala prueba: se ejecutó como `root` del clon (no es usuario de AD, sin ticket) | Probar con usuarios reales: `smbclient //localhost/<share> -U "CORP\\user%pass" -c ls` |
| 26 | `zsh: event not found: Lab` | `!` dentro de comillas dobles: expansión del historial de zsh | Comillas simples por fuera y dobles por dentro (`'echo "…!…" \| kinit …'`) |
| 27 | `net use` → error 1219 (múltiples conexiones) y `dir … /delete` no cerraba la conexión | Había una conexión abierta con otro usuario; además se tecleó `\delete` y `dir` en lugar de `net use … /delete` | `net use \\10.0.1.12\ventas /delete` (con barra normal) antes de nuevos intentos |
| 28 | Sin eventos de fallo en el DC; el filtro `Keywords=4503599627370496` no devolvía nada | El DC solo auditaba éxito en `Credential Validation`; además el valor de `Keywords` era erróneo | `auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable`; filtrar con `KeywordsDisplayNames` (`Audit Failure`) |
| 29 | `pdias` bloqueado tras 2 intentos fallidos desde el DC | El cliente SMB de Windows reintenta la autenticación: 5 eventos 4776 en 15 s, umbral de bloqueo = 5 | Se aprovechó como material del ticket 2. **Regla:** con umbral 5, hacer a lo sumo 1 o 2 intentos fallidos por usuario |
| 30 | Errores al teclear en la consola del DC (`Keywords4503…` sin `=`, `\delete`) | Comandos largos tecleados a mano | Dar los comandos del DC en **líneas cortas**, una por una |

---

## 11. Decisiones tomadas (no reabrir sin razón)

- Interfaz **`wg-hybrid`**, rango `10.91.0.0/24`; `wg0` y `wg-azure` intactos.
- Gateway con IP fija en AWS; Wonderland inicia el túnel (CGNAT).
- Túnel levantado **a mano**, sin habilitar el servicio permanente.
- Clientes **Ubuntu** (`realmd/sssd`), no Windows. Samba como *member server*.
- Stack base CloudFormation **intacto**; clientes por consola/AMI, no por CFN.
- Contraseña común del lab `Wonderland#2026!Lab` (solo lab; no reutilizar).
- Ticket 5 reformulado: las GPO no aplican a Ubuntu → "usuario sin acceso por política de depto" (`simple_allow_groups`/`realm permit` y grupo faltante).
- Subida de RAM de la VM a 9 GB: **diferida al Nivel 2** (el 10-10 el host tenía 15 GB, 12 usados y 3,2 GB libres; ver F.3).
- Monitoreo: v2.
- Clones desde Ubuntu 24.04 + user data (la AMI propia no es lanzable en el sandbox).
- Join con `--membership-software=samba`; `smb.conf` se escribe a mano en el paso de Samba.
- Clones con el DC como único DNS (el DNS de la VPC no conoce el dominio hasta el Nivel 2).
- DC con zona horaria de Chile y NTP externo (`0.pool.ntp.org`).
- (10-10) Samba con `winbind` además de sssd; share `0777` y control real por `valid users` (E.3).
- (10-10) Comandos a los clones por **SSH desde el host** a través del túnel (`AD.pem`), no por SSM.
- (10-10) Política de contraseñas de dominio: `MinPasswordLength` 8 en la Default Domain Policy (las GPO de contraseña enlazadas a OU no aplican a cuentas de dominio).
- (10-10) Hook de libvirt: reinserta las reglas de ambos bridges al arrancar `pentest-lab` o `lab-hybrid`.
- (10-10) Limpieza de AWS (AMI, snapshots, stack) no se hace a mano: el sandbox lo elimina al terminar el tiempo o al cerrarlo.

---

## 12. Plan para terminar el Nivel 1

Resumen (los pasos se dan de a uno en el chat):

1. ~~Confirmar el script y crear la AMI~~ (hecho; AMI inutilizable, ver 9.2).
2. ~~Lanzar los clones~~ (IT verificado; Ventas, RRHH y Gerencia lanzados).
3. ~~Correr `dept-config.sh` en Ventas, RRHH y Gerencia~~ (hecho el 10-10, Anexo E).
4. ~~Verificar dominio~~ (hecho, E.1 y F.2).
5. ~~Samba por departamento~~ (hecho, E.3 y E.5).
6. ~~Simulación de tráfico~~ (hecho, E.3).
7. ~~5 tickets N1~~ (hechos, F.1).
8. ~~Cierre~~ (hecho, F.2 y F.3). Lo que sigue está en F.4.

---

## Anexo A — Fases A-C originales (bitácora de ejecución)

Resultado: VM `win-dc01` (6 GB / 4 vCPU / 60 GB en USB ext4), Windows Server 2022 Standard Desktop Experience, dominio `corp.wonderland.local` operativo con 4 OUs, 4 usuarios y 3 GPOs. Cuatro causas raíz de infraestructura, ninguna de AD en sí.

### A.1 Causa raíz #1 — Driver NetKVM no se instala solo
- **Síntoma:** la NIC virtio aparece en Device Manager como "Ethernet Controller" (Other devices) con warning; la búsqueda automática pide internet, que no existe sin el driver (problema circular).
- **Solución:** Device Manager → clic derecho → Update driver → **Browse my computer for drivers** (nunca la opción automática) → carpeta `NetKVM\2k22\amd64` del ISO virtio-win montado como segundo CD-ROM. Instala "Red Hat VirtIO Ethernet Adapter".

### A.2 Causa raíz #2 — Permisos del USB en cada remount
- **Síntoma:** `Cannot access storage file ... Permission denied (as uid:950, gid:950)` al arrancar la VM, aunque `chown 950:950` sobre el `.qcow2` ya se había aplicado.
- **Causa:** qemu (`libvirt-qemu`, uid/gid 950) necesita el bit `x` en **cada directorio de la ruta**. `/run/media/diego/` (gestionado por `udisks2`) queda en `700` tras cada remount.
- **Solución (repetir al reconectar el USB):**
```bash
sudo chown 950:950 /run/media/diego/LABDISK/win-dc01.qcow2
sudo chmod o+x /run/media/diego/
sudo chmod o+x /run/media/diego/LABDISK/
```
- **Pendiente:** regla `udev` que lo automatice.

### A.3 Causa raíz #3 — DNS sin resolver con IP funcionando
- **Síntoma:** `ping 8.8.8.8` OK; `ping google.com` y `nslookup` fallan. Firewall de Windows descartado (se probó desactivado). Docker generó sospecha falsa; el host usa `ufw`.
- **Causa:** `ufw` con `Default: deny (routed)` y reglas `ALLOW FWD` solo para `virbr0` y `docker0`, ninguna para `virbr-lab`. El tráfico nuevo (DNS UDP) se descartaba; el ping pasaba por conntrack de una conexión ya establecida.
- **Solución:**
```bash
sudo ufw route allow in on virbr-lab out on wlp0s20u2
sudo ufw route allow in on wlp0s20u2 out on virbr-lab
sudo ufw reload
```
- **Regla permanente:** toda red libvirt custom nueva necesita su par de `ufw route allow`.

### A.4 Causa raíz #4 — PowerShell (x86) en vez de x64
- **Síntoma:** `Install-WindowsFeature`, `Import-Module ServerManager` e `Import-Module ADDSDeployment` fallan ("term not recognized"/"module not found"); `Test-Path` de la carpeta del módulo da `False`; `sfc /scannow` falla con "could not start the repair service".
- **Causa:** la consola era Windows PowerShell (x86), que busca módulos en `C:\Windows\SysWOW64\WindowsPowerShell\v1.0\Modules\`, donde no existen las herramientas de administración. Se identifica solo por "(x86)" en el título.
- **Solución:** abrir "Windows PowerShell" (sin "(x86)") como Administrador. Alternativa sin el módulo: `dism /online /enable-feature /featurename:DirectoryServices-DomainController /all` (instala el rol, no reemplaza `ADDSDeployment` para `Install-ADDSForest`).

### A.5 Promoción y verificación
```powershell
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "corp.wonderland.local" -DomainNetbiosName "CORP" -InstallDns:$true -SafeModeAdministratorPassword (ConvertTo-SecureString "<password-dsrm>" -AsPlainText -Force)
```
Reinició y el login fue `CORP\Administrator`. `Get-ADDomain`: `DomainMode Windows2016Domain`, FSMO (`PDCEmulator`, `RIDMaster`, `InfrastructureMaster`) en `WIN-DC01`; `nslookup corp.wonderland.local` → `192.168.50.10`. `sfc /scannow` en la consola correcta: sin violaciones de integridad (los fallos previos eran por la consola x86).

`dcdiag /v`: pasaron Connectivity, Advertising, FrsEvent, KccEvent, KnowsOfRoleHolders, MachineAccount, NCSecDesc, NetLogons, ObjectsReplicated, Replications, RidManager, Services, VerifyReferences, CrossRefValidation (5 particiones), LocatorCheck, Intersite.
- **SystemLog:** falló en la primera corrida por ruido transitorio (Credential Guard, timeouts de `edgeupdate`, permisos COM, un fallo puntual de WinRM al crear SPNs); en el retest pasó limpio.
- **DFSREvent:** `Error 1355: The specified domain either does not exist or could not be contacted`, con reintentos cada ~60 min. Esperable en un DC único mientras DFSR/SYSVOL termina de inicializar; no bloqueante. Se resuelve solo o al agregar el segundo DC (Nivel 2).

### A.6 Fase C — OUs, usuarios y GPOs
OUs `IT`, `Ventas`, `RRHH`, `Gerencia` bajo `DC=corp,DC=wonderland,DC=local` (verificadas con `Get-ADOrganizationalUnit`). Usuarios importados por CSV con `New-ADUser`, contraseña inicial `Password123!` y `ChangePasswordAtLogon $true`:

| Usuario | OU |
|---|---|
| Juan Perez (`jperez`) | IT |
| Maria Soto (`msoto`) | Ventas |
| Pedro Dias (`pdias`) | RRHH |
| Ana Rojas (`arojas`) | Gerencia |

> [!warning] La guía Nivel 1 está desactualizada en este punto
> Su CSV dice `Pedro,Diaz,RRHH,pdiaz`. El usuario real es `pdias`.

> [!note] Backticks en PowerShell
> Un espacio extra después del backtick de continuación rompe el parseo. Se resolvió reescribiendo el comando en una sola línea.

GPOs (creadas y linkeadas por PowerShell, contenido editado en `gpmc.msc`):
```powershell
Import-Module GroupPolicy
New-GPO -Name "Politica-Contrasenas-Corp" | New-GPLink -Target "DC=corp,DC=wonderland,DC=local"
New-GPO -Name "Fondo-Escritorio-Corp" | New-GPLink -Target "DC=corp,DC=wonderland,DC=local"
New-GPO -Name "Restriccion-PanelControl-Ventas-RRHH"
New-GPLink -Name "Restriccion-PanelControl-Ventas-RRHH" -Target "OU=Ventas,DC=corp,DC=wonderland,DC=local"
New-GPLink -Name "Restriccion-PanelControl-Ventas-RRHH" -Target "OU=RRHH,DC=corp,DC=wonderland,DC=local"
```

| GPO | Ruta en el editor | Configuración |
|---|---|---|
| Politica-Contrasenas-Corp | Computer Config → Windows Settings → Security Settings → Account Policies → Password Policy | Longitud mínima 8, complejidad habilitada |
| Fondo-Escritorio-Corp | User Config → Administrative Templates → Desktop → Desktop | Desktop Wallpaper habilitado, imagen de prueba, estilo Fill |
| Restriccion-PanelControl-Ventas-RRHH | User Config → Administrative Templates → Control Panel | Prohibit access to Control Panel and PC settings: Enabled |

Verificación: `gpupdate /force` y revisión en el propio DC para las dos GPO de dominio. La tercera no es verificable, porque el DC vive en "Domain Controllers" y los clientes AWS son Ubuntu: **las GPO no aplican a los clientes de este lab** (de ahí el ticket 5 reformulado).

> [!warning] Posible solapamiento de políticas de contraseña (a verificar)
> `Politica-Contrasenas-Corp` está linkeada al dominio como GPO normal, y el lockout se fijó después con `Set-ADDefaultDomainPasswordPolicy`. La política de cuentas de dominio efectiva la manda la GPO de mayor precedencia en el dominio, que probablemente sigue siendo Default Domain Policy. **Confirmado el 2026-10-08:** `Get-ADDefaultDomainPasswordPolicy` devuelve `MinPasswordLength = 7` y `LockoutThreshold = 5`; la longitud mínima vigente es 7, no 8.

---

## Anexo B — Lo que traía el handoff y no estaba arriba

### B.1 Estructura del proyecto
- **Parte 1 (Nivel 1):** DC on-prem, red AWS, túnel WireGuard, clientes Ubuntu unidos al dominio, 5 tickets N1. Es lo publicable como versión 1.
- **Parte 2 (Nivel 2, a futuro):** segundo DC, DNS híbrido nativo (Route 53 Resolver), FGPP, LAPS, hardening (SMBv1, NTLM, auditoría avanzada). Se empieza solo con el Nivel 1 cerrado y verificado. Guía de referencia: `nivel2-hybrid-ad-lab-completo.md`.

### B.2 Decisiones del handoff adicionales
- DC: Windows Server 2022 **Standard**, Desktop Experience (ni Core ni Datacenter).
- VPN: WireGuard y no Site-to-Site clásico, por CGNAT; el gateway con IP fija va en AWS y Wonderland inicia la conexión.
- CloudFormation (no Terraform) solo para red + gateway; los clientes se configuran a mano "como práctica".
- Subnet privada sin NAT Gateway administrado (ahorro ~USD 32/mes): el gateway EC2 hace de router.
- **Pendiente:** VPC Endpoints de SSM en vez de salida a internet directa por el gateway (más seguro, evita depender de la ruta `0.0.0.0/0` de la subnet privada).
- Administración AWS solo por SSM Session Manager, sin SSH/RDP expuesto.
- Storage de la VM en disco USB **ext4**, no exFAT (sparse files para qcow2 y permisos Unix), montado en `/run/media/diego/LABDISK/`.
- RAM de la VM: 6 GB / 4 vCPU. **Subir a 9 GB** después de cerrar el Nivel 1 (decisión tomada, ejecución diferida).

### B.3 Preferencias de trabajo
- Comandos: solo el comando, sin preámbulo ni explicación salvo que se pida.
- Directo, sin confirmaciones innecesarias cuando la intención es clara.
- Español para toda la documentación.
- Sin cuadros/tarjetas de pasos (step_card): texto plano.
- Antes de asumir causas exóticas, descartar lo obvio: consola y versión usada, cómo se pegó el comando, estado de la VM (`virsh list --all`), permisos en toda la cadena de directorios.

### B.4 Archivos del proyecto (tras esta consolidación)
- `ad-hybrid-lab-documento-final-fases-0-3.md` — este documento (reemplaza al handoff y a la bitácora A-C).
- `nivel1-hybrid-ad-lab-intermedio.md` — guía original del Nivel 1: **obsoleta**, su contenido vigente está en el Anexo C; puede archivarse o borrarse.
- `nivel2-hybrid-ad-lab-completo.md` — guía de la Parte 2.
- `hybrid-ad-lab-network-v2.yaml` — template CloudFormation desplegado.
- `referencia-operativa-maestra.md` — referencia viva, sección 11 del AD.

---

## Anexo C — Referencia de reconstrucción (rescatado de la guía Nivel 1, ya corregido)

Con esto el documento no depende de `nivel1-hybrid-ad-lab-intermedio.md`.

### C.1 Prerrequisitos
- ISO Windows Server 2022 (Evaluation, Desktop Experience) e ISO **virtio-win** (sin sus drivers el instalador no ve el disco virtual).
- `wireguard-tools` en el host (`sudo pacman -S wireguard-tools`).
- RAM libre suficiente para el DC (hoy 6 GB; objetivo 9 GB tras cerrar el Nivel 1).
- CIDR sin solapar: `192.168.50.0/24` (lab), tu LAN doméstica real y `10.0.0.0/16` (VPC). Revisar con `ip route` antes de fijarlos.

### C.2 Red libvirt dedicada (no usar la NAT default `virbr0`)
```bash
cat > /tmp/lab-net.xml << 'XML'
<network>
  <name>lab-hybrid</name>
  <forward mode="nat"/>
  <bridge name="virbr-lab" stp="on" delay="0"/>
  <ip address="192.168.50.1" netmask="255.255.255.0">
    <dhcp>
      <range start="192.168.50.100" end="192.168.50.200"/>
    </dhcp>
  </ip>
</network>
XML
sudo virsh net-define /tmp/lab-net.xml
sudo virsh net-start lab-hybrid
sudo virsh net-autostart lab-hybrid
```
Después: el par de `ufw route allow` de A.3 y el bloque `lab-hybrid` del hook de libvirt.

### C.3 Creación de la VM (referencia)
> [!note] Valores reales distintos de la guía
> La guía usaba disco de 80 GB en `/var/lib/libvirt/images/`. La VM real tiene **60 GB en `/run/media/diego/LABDISK/win-dc01.qcow2`**. No se conserva el comando exacto que se usó; este es el equivalente.

```bash
sudo virt-install \
  --name win-dc01 \
  --memory 6144 --vcpus 4 \
  --disk path=/run/media/diego/LABDISK/win-dc01.qcow2,size=60,bus=virtio \
  --cdrom /ruta/a/WinServer2022.iso \
  --disk /ruta/a/virtio-win.iso,device=cdrom \
  --os-variant win2k22 \
  --network network=lab-hybrid,model=virtio \
  --graphics spice --video qxl
```

### C.4 Instalación de Windows y red del DC
1. Elegir **Windows Server 2022 Standard (Desktop Experience)**, no Core.
2. En "dónde instalar", si no aparece el disco: Load driver → CD-ROM virtio-win → `viostor\2k22\amd64`.
3. Definir la contraseña de Administrator.
4. NIC: si no la detecta, cargar `NetKVM\2k22\amd64` por "Browse my computer" (ver A.1).
5. IP estática `192.168.50.10/24`, gateway `192.168.50.1`, DNS `127.0.0.1` (tras ser DC se apunta a sí mismo).
6. Renombrar a `WIN-DC01`, reiniciar y correr Windows Update **antes** de promover a DC.

### C.5 Usuarios por lote (versión corregida)
`usuarios.csv`:
```csv
Nombre,Apellido,OU,Usuario
Juan,Perez,IT,jperez
Maria,Soto,Ventas,msoto
Pedro,Dias,RRHH,pdias
Ana,Rojas,Gerencia,arojas
```
Importación (en una sola línea por el problema de backticks; con `-Enabled $true`, y la contraseña común actual en vez de `Password123!`):
```powershell
$p = ConvertTo-SecureString "Wonderland#2026!Lab" -AsPlainText -Force; $d="DC=corp,DC=wonderland,DC=local"; Import-Csv .\usuarios.csv | ForEach-Object { New-ADUser -Name "$($_.Nombre) $($_.Apellido)" -SamAccountName $_.Usuario -UserPrincipalName "$($_.Usuario)@corp.wonderland.local" -Path "OU=$($_.OU),$d" -AccountPassword $p -ChangePasswordAtLogon $false -Enabled $true }
```

### C.6 Diseño AWS (principios de la guía)
- Subnet pública solo para el gateway; subnet privada sin IP pública para los clientes.
- SG del gateway: solo `UDP/51820` entrante desde `0.0.0.0/0` (inevitable por CGNAT).
- SG privado: solo tráfico desde el rango on-prem/túnel y entre miembros, con los puertos de AD.
- Administración solo por SSM Session Manager. (Detalle real en las secciones 4 y 6.)

### C.7 Tickets N1 (versión vigente, con clientes Ubuntu)

| # | Ticket | Cómo romperlo | Cómo resolverlo | Verificación |
|---|---|---|---|---|
| 1 | Usuario olvidó su contraseña | Cambiar la contraseña de `msoto` en el DC | `Set-ADAccountPassword -Identity msoto -Reset -NewPassword ...` o ADUC | `kinit msoto@CORP.WONDERLAND.LOCAL` |
| 2 | Cuenta bloqueada | 5 `kinit` con contraseña errónea a `arojas` | `Unlock-ADAccount arojas`; revisar Event ID 4740 | login OK |
| 3 | Cliente no resuelve el dominio | `sudo wg-quick down wg-hybrid` en el host, o editar el DNS del netplan del cliente | Levantar el túnel / restaurar `60-ad-dns.yaml`; `resolvectl status`, `dig SRV` | `realm list`, `dig +short -t SRV _ldap._tcp.corp.wonderland.local` |
| 4 | Usuario nuevo con cuenta y acceso a la carpeta de su depto | Crear usuario sin grupo | `New-ADUser` en la OU + `Add-ADGroupMember GG_<dept>` + permisos del share Samba | `smbclient` con el usuario nuevo |
| 5 | (Reformulado) Usuario sin acceso por política de depto. El original, "GPO de fondo no aplica", no sirve porque las GPO no aplican a Ubuntu | Quitar al usuario de `GG_<dept>` | Volver a agregarlo; revisar `realm permit`/`simple_allow_groups`, esperar caché de sssd (`sudo sss_cache -E`) | login OK |

Cada ticket se documenta con: síntoma, diagnóstico, comando, verificación y captura.

### C.8 Checklist final del Nivel 1 (versión vigente; cumplido el 2026-10-10, resultados en F.2)
- [x] `dcdiag /v` sin errores relevantes (DFSREvent puede seguir en reintento, ver A.5).
- [x] 4 clones unidos y sus cuentas de equipo en las OUs correspondientes (`Get-ADComputer`).
- [x] `wg show` con handshake reciente en ambos lados.
- [x] Matriz de acceso por departamento comprobada (entra el del depto y `GG_IT`, no entran los demás).
- [x] Share Samba por departamento accesible solo para su grupo.
- [x] Simulación de tráfico ejecutada y capturada.
- [x] Los 5 tickets resueltos y documentados.
- [x] Equipos apagados y EIP confirmada (ver C.9).

### C.9 Costos y apagado
- Apagar clones y modelo cuando no se trabaje; el gateway t3.micro puede quedar encendido.
- Una EIP sin asociar cobra: confirmar que sigue asociada al gateway, o liberarla al cerrar el lab.
- Considerar un script `start`/`stop` si el lab se vuelve rutina de varias sesiones.

## Anexo D — Pasos de la sesión siguiente (planificados para 2026-10-09; ejecutados el 2026-10-10, ver Anexos E y F)

> [!note] El paso 1 (permisos del USB) se reemplazó el 10-10 por la entrada fstab de F.3; los `chmod` siguen siendo el plan B. El paso 13 (borrar AMI y snapshot) no hizo falta: el sandbox se reinició y los eliminó.

Cada comando indica dónde se ejecuta. Un paso a la vez.

**Arranque (Wonderland, terminal del host)**
1. Montar el USB y dar permisos: `sudo mkdir -p /run/media/diego/LABDISK && mountpoint -q /run/media/diego/LABDISK || sudo mount LABEL=LABDISK /run/media/diego/LABDISK` y `sudo chmod o+x /run/media /run/media/diego /run/media/diego/LABDISK`; si libvirt dice `Permission denied (uid:950)`: `sudo chown 950:950 /run/media/diego/LABDISK/win-dc01.qcow2`.
2. Red y VM: `sudo virsh net-start lab-hybrid` (si no está activa), `sudo virsh start win-dc01` (si está paused: `virsh resume`).
3. Túnel: `sudo wg-quick up wg-hybrid`; verificar `sudo wg show wg-hybrid` (handshake) y `ping -c2 10.91.0.1`.
4. Consola AWS (Learner Lab): confirmar sesión vigente, EIP asociada al gateway, 5 instancias `running` 3/3 (gateway, ubu-it-01, ventas, rrhh, gerencia) y `Dept` correcto. Si el lab se reinició: handshake nuevo, EIP y perfil `LabInstanceProfile` (adjuntar + reboot).
5. DC (PowerShell 64 bits en la consola): `Get-Date` y `w32tm /query /status` (hora Chile, sincronizada).

**Clones (SSM, uno por uno: Ventas, RRHH, Gerencia)**
6. Ajuste DNS solo DC + verificación: ver 9.1 (sed) y `grep -n DC_IP /usr/local/bin/dept-config.sh`.
7. `sudo dept-config.sh <dept>` (pide la password de Administrator); debe terminar sin `AVISO`.
8. Verificar: `realm list`, `id <usuario>@corp.wonderland.local`, `kinit`, y en el DC `Get-ADComputer -Filter * | select Name,DistinguishedName` (4 equipos en su OU).
9. Matriz de acceso (contraseña `Wonderland#2026!Lab`).

**Samba, tráfico, tickets, cierre**
10. Samba en IT, luego replicar (smb.conf manual, `/srv/shares/<dept>`, `valid users = @CORP\GG_<dept>`; probar con smbclient, incluido el rechazo cruzado).
11. Tráfico simulado + `tcpdump -ni wg-hybrid` en el host.
12. 5 tickets N1 con captura.
13. Cierre: checklist, apagar EC2/DC, `wg-quick down wg-hybrid`, EIP, borrar AMI `ubu-ad-model` + snapshot, `<clock offset='localtime'>` en la VM, decidir GPO 7 vs 8, actualizar docs.

**Template**: el archivo `hybrid-ad-lab-network.yaml` del repositorio es la **v3** (clientes opcionales con `DeployClients`), validada en un despliegue real el 10-10 (Anexo E).


---

## Anexo E — Sesión del 2026-10-10: reconstrucción, clones, Samba y tráfico

Cubre desde "retomamos" hasta el inicio de los tickets: cómo se levantó todo el stack desde cero tras el reinicio del sandbox, qué falló y cómo se resolvió. Los errores 18 a 30 están en la tabla de la sección 10.

> [!warning] Dependencia del sandbox
> Si el sandbox de AWS se reinicia, desaparecen el stack y las instancias (el DC, en el host, sobrevive). Pasó entre el 8 y el 10 de octubre: por eso el stack se recreó (`ActiveDirectoryLab`) y las claves WireGuard del server cambiaron.

### E.0 IDs de la sesión (efímeros)

| Pieza | Valor |
|---|---|
| Stack | `ActiveDirectoryLab` (template v3; luego update con `DeployClients=true`) |
| Gateway | `i-04d42803101894d9e`, EIP `<EIP-GATEWAY>`, `LabInstanceProfile` asignado y reiniciado |
| VPC / subnet pública / subnet privada / SG privado | `vpc-09955c8d422b4f3d6` / `subnet-0bf0af9c74155ecf5` / `subnet-0f5e5f89c7f547f35` / `sg-05bdcdcc903205d3f` |
| Clave pública del host (`wg-hybrid`) | `<CLAVE-PUBLICA-HOST>` |
| Clave pública del server (nueva) | `<CLAVE-PUBLICA-SERVER>` |
| Clones | `ubu-it-01` .11, `ubu-ventas-01` .12, `ubu-rrhh-01` .13, `ubu-gerencia-01` .14: los 4 unidos, cada uno en su OU |
| Matriz de acceso | IT: jperez · Ventas: jperez+msoto · RRHH: jperez+pdias · Gerencia: jperez+arojas |

### E.1 Cronología (pasos ejecutados)

| # | Dónde | Qué |
|---|---|---|
| 1 | Host | Permisos del USB: `chmod o+x /run/media /run/media/diego /run/media/diego/LABDISK`; `chown 950:950` al qcow2 |
| 2 | Host | `virsh net-list` / `virsh list`: `lab-hybrid` activa, `win-dc01` apagada |
| 3 | Host | `virsh start win-dc01`, `wg-quick up wg-hybrid` |
| 4 | AWS | El ping a `10.91.0.1` no respondía: el stack no existía (ver error 18). Se redesplegó |
| 5 | AWS | Deploy del stack con el template v3. Parámetros: `WgClientPublicKey` = clave del host, `KeyName=AD`, `DeployClients=false` |
| 6 | AWS | El gateway no instaló WireGuard (error 19). Template corregido; stack borrado y recreado |
| 7 | AWS | Rol `LabInstanceProfile` al gateway + reboot |
| 8 | AWS (SSM) | Clave pública del server: `sudo cat /etc/wireguard/server_public.key` |
| 9 | Host | `wg-hybrid.conf`: `PublicKey` y `Endpoint` nuevos; `wg-quick up`; handshake OK |
| 10 | AWS | Update del stack con `DeployClients=true` (direct update, plantilla existente; error 20) |
| 11 | AWS | Rol `LabInstanceProfile` a los 4 clones; esperar ~10 min (instalación del user data); reboot |
| 12 | SSM | `ubu-it-01`: `sudo dept-config.sh` (falló por el reject de libvirt, error 21); se corrigió y se unió |
| 13 | SSM | Ventas, RRHH, Gerencia: `ls /var/lib/clone-ready && sudo dept-config.sh` |
| 14 | DC | `Get-ADComputer`: 4 equipos en su OU |
| 15 | Host (SSH) | Matriz de acceso con `sssctl user-checks` en los 4 clones, por SSH a través del túnel |
| 16 | Host (SSH) | Samba en IT (errores 23 a 25), luego réplica a los otros 3 con `/tmp/smbsetup.sh` |
| 17 | Host (SSH) | Tráfico con `/tmp/traffic.sh` + `tcpdump` en `wg-hybrid` |
| 18 | DC | Tráfico del DC hacia el share de Ventas; fallos de contraseña (errores 27 y 28) |
| 19 | DC | Auditoría de fallos activada; bloqueo accidental de `pdias` (error 29); evento 4740 capturado |
| 20 | Host (SSH) | `kinit pdias` con contraseña correcta: `Client's credentials have been revoked` (síntoma del ticket 2) |

**Cambio de método:** a partir del paso 15 los comandos a los clones van por **SSH desde el host** (`ssh -i AD.pem ubuntu@10.0.1.x`), no por SSM. El SG permite el puerto 22 desde el rango del túnel. Es mucho más rápido que abrir sesiones SSM.

### E.2 Errores de la sesión

Filas 18 a 30 de la sección 10.

### E.3 Decisiones y hallazgos de la sesión

- **Samba con `winbind`, no solo sssd.** `smb.conf` permanente: `security = ads`, `workgroup = CORP`, idmap `rid` (10000-999999), `disable netbios = yes`, `smb ports = 445`. sssd y winbind conviven: el login de usuarios sigue por sssd (UIDs `3614…`), el acceso al share lo valida winbind (UIDs `1xxxx`).
- **Permisos del share:** carpeta `0777` en `/srv/shares/<dept>`; el control real es `valid users = @"CORP\GG_<Dept>"`. Aceptable para el lab; en producción se endurecerían los permisos de sistema de archivos.
- **`realm join` con Samba no admite `AD_PASS` no interactivo** (la contraseña se pide dos veces: una por el script, otra por `realm`).
- **El SMB entre clones no cruza el túnel** (va por la VPC): en la captura de `wg-hybrid` solo aparecen 6 paquetes del puerto 445. El SMB que sí cruza el túnel es el del DC hacia los clones.
- **Captura de evidencia** (`ad-lab.pcap`): puerto 53 → 52 paquetes · 88 → 366 · 389 → 185 · 445 → 6.
- **Template v3 validado en un despliegue real** (gateway corregido + 4 clientes con user data). Lo único sin probar es el `stty -echo`.
- **Evento 4740** (`pdias`, 13:04:20): `Caller Computer Name = \\WIN-DC01`, porque el DC es quien valida la contraseña cuando falla un cliente SMB.

### E.4 Reconstrucción rápida (si el sandbox se reinicia)

1. **Host:** permisos del USB, `virsh start win-dc01`, `net-start lab-hybrid` si hace falta. Insertar de nuevo las reglas `nft` (sección 2, error 21) si hace falta.
2. **AWS:** crear el stack con `hybrid-ad-lab-network-v3.yaml` (`WgClientPublicKey` = clave del host, `KeyName=AD`, `DeployClients=false`); asignar `LabInstanceProfile` al gateway y reiniciar.
3. **SSM (gateway):** `sudo cat /etc/wireguard/server_public.key`.
4. **Host:** actualizar `PublicKey` y `Endpoint` en `/etc/wireguard/wg-hybrid.conf`; `wg-quick up wg-hybrid`.
5. **AWS:** update directo del stack con `DeployClients=true`; `LabInstanceProfile` a los 4 clones; esperar ~10 min; reiniciar.
6. **SSH desde el host** (o SSM): `ls /var/lib/clone-ready && sudo dept-config.sh` en cada clon (la contraseña de `Administrator` se pide dos veces).
7. **Host:** Samba por clon con el script de abajo.

### E.5 Scripts usados (para rehacerlos)

`/tmp/smbsetup.sh` (se corre con `ssh -i AD.pem ubuntu@10.0.1.12 'bash -s -- ventas GG_Ventas msoto pdias' < /tmp/smbsetup.sh`; argumentos: share, grupo, usuario que debe entrar, usuario que debe ser rechazado):

```bash
set -e
SHARE=$1; GRP=$2; OKU=$3; NOU=$4
export DEBIAN_FRONTEND=noninteractive
sudo apt-get install -y winbind libnss-winbind >/dev/null 2>&1
sudo sed -i -E '/^(passwd|group):/{/winbind/! s/$/ winbind/}' /etc/nsswitch.conf
WG=$(adcli info corp.wonderland.local | awk '/domain-short/{print $3}')
sudo mkdir -p /srv/shares/$SHARE && sudo chmod 0777 /srv/shares/$SHARE
sudo tee /etc/samba/smb.conf >/dev/null << EOS
[global]
   workgroup = $WG
   realm = CORP.WONDERLAND.LOCAL
   security = ads
   kerberos method = secrets and keytab
   idmap config * : backend = tdb
   idmap config * : range = 3000-7999
   idmap config $WG : backend = rid
   idmap config $WG : range = 10000-999999
   disable netbios = yes
   smb ports = 445

[$SHARE]
   path = /srv/shares/$SHARE
   read only = no
   valid users = @"$WG\\$GRP"
   create mask = 0660
   directory mask = 0770
EOS
sudo testparm -s >/dev/null 2>&1 && echo "smb.conf OK"
sudo systemctl enable --now winbind smbd >/dev/null 2>&1
sudo systemctl restart winbind smbd; sleep 3
sudo net ads testjoin
sudo wbinfo -t | tail -1
for u in $OKU $NOU; do echo "== $u"; smbclient //localhost/$SHARE -U "$WG\\$u%Wonderland#2026!Lab" -c 'ls' 2>&1 | grep -E "blocks|NT_STATUS" | head -2; done
```

`/tmp/traffic.sh` (argumentos: usuario, share propio, IP del clon ajeno, share ajeno):

```bash
U=$1; OWN=$2; FIP=$3; FSH=$4; PW='Wonderland#2026!Lab'; D=corp.wonderland.local
for i in 1 2 3; do
  dig +short SRV _ldap._tcp.$D >/dev/null
  dig +short SRV _kerberos._tcp.$D >/dev/null
  echo "$PW" | kinit $U@${D^^} >/dev/null 2>&1 && klist >/dev/null 2>&1; kdestroy 2>/dev/null
  echo "bad" | kinit usuario.falso@${D^^} >/dev/null 2>&1
  ldapsearch -x -H ldap://win-dc01.$D -D "$U@$D" -w "$PW" -b DC=corp,DC=wonderland,DC=local "(sAMAccountName=$U)" dn >/dev/null 2>&1
  smbclient //localhost/$OWN -U "CORP\\$U%$PW" -c 'ls' >/dev/null 2>&1
  smbclient //$FIP/$FSH -U "CORP\\$U%$PW" -c 'ls' >/dev/null 2>&1
  sleep 2
done
echo "$(hostname -s): OK"
```

Regla de `nft` (en el host, error 21):

```bash
sudo nft insert rule ip libvirt_network guest_input oifname "virbr-lab" ip saddr 10.0.0.0/16 accept
sudo nft insert rule ip libvirt_network guest_input oifname "virbr-lab" ip saddr 10.91.0.0/24 accept
```

Matriz de acceso (desde el host, por SSH):

```bash
for ip in 11 12 13 14; do ssh -i AD.pem -o ConnectTimeout=8 ubuntu@10.0.1.$ip 'echo "##### $(hostname -s)"; for u in jperez msoto pdias arojas; do printf "%s: " $u; sudo sssctl user-checks $u@corp.wonderland.local -s sshd -a acct 2>&1 | grep "pam_acct_mgmt:"; done'; done
```

> [!note] Actualización posterior
> El paso 1 de la reconstrucción ("insertar de nuevo las reglas `nft`") ya no es necesario tras el parche del hook de libvirt (F.3): las reglas se reinsertan solas cuando arranca `pentest-lab` o `lab-hybrid`. Si las redes ya estaban activas, se pueden reaplicar a mano con `sudo /etc/libvirt/hooks/network lab-hybrid started begin -`.


---

## Anexo F — Cierre del Nivel 1 (2026-10-10, tarde): tickets N1 y ajustes finales

Los resultados que en la sesión se mostraron como capturas se transcriben aquí como comandos y salidas relevantes.

### F.1 Tickets N1

Plantilla: síntoma → diagnóstico → comando → verificación.

#### Ticket 1 — Contraseña olvidada (`msoto`)

- **Síntoma:** `msoto` no puede autenticarse (había acumulado 3 fallos).
- **Resolución (DC, PowerShell, línea por línea):**

```powershell
$p = ConvertTo-SecureString 'Nueva#2026!Lab' -AsPlainText -Force
Set-ADAccountPassword msoto -Reset -NewPassword $p
Get-ADUser msoto -Properties PasswordLastSet,LockedOut | Select SamAccountName,PasswordLastSet,LockedOut
```

```
SamAccountName PasswordLastSet       LockedOut
msoto          10/10/2026 1:20:08 PM False
```

- **Verificación (host, a `ubu-ventas-01`):**

```bash
ssh -i AD.pem ubuntu@10.0.1.12 'echo "Nueva#2026!Lab" | kinit msoto@CORP.WONDERLAND.LOCAL && klist'
```

```
Default principal: msoto@CORP.WONDERLAND.LOCAL
Valid starting     Expires            Service principal
10/10/26 16:21:21  10/11/26 02:21:21  krbtgt/CORP.WONDERLAND.LOCAL@CORP.WONDERLAND.LOCAL
```

> [!note] Hora de los clones
> Los clones están en UTC (16:21 UTC = 13:21 en Chile, UTC-3) y el DC en hora de Chile: la diferencia es solo de zona, no de reloj, y Kerberos no se queja.

#### Ticket 2 — Cuenta bloqueada (`pdias`)

- **Causa:** un acceso SMB desde el DC con contraseña errónea. El cliente SMB de Windows reintenta: 5 eventos 4776 en 15 s, con umbral de bloqueo 5. Ver error 29.
- **Síntoma (clon `ubu-rrhh-01`):** `kinit pdias` con la contraseña correcta → `Client's credentials have been revoked`.
- **Diagnóstico (DC):** evento 4740.

```
TimeCreated : 10/10/2026 1:04:20 PM
Message     : A user account was locked out.
Account Name: pdias
Caller Computer Name: \\WIN-DC01
```

- **Resolución y verificación (DC):**

```powershell
Unlock-ADAccount pdias
Get-ADUser pdias -Properties LockedOut | Select SamAccountName,LockedOut
```

```
SamAccountName LockedOut
pdias          False
```

- **Verificación (host):**

```bash
ssh -i AD.pem ubuntu@10.0.1.13 'echo "Wonderland#2026!Lab" | kinit pdias@CORP.WONDERLAND.LOCAL && klist'
```

```
Default principal: pdias@CORP.WONDERLAND.LOCAL
10/10/26 16:18:29  10/11/26 02:18:29  krbtgt/CORP.WONDERLAND.LOCAL@CORP.WONDERLAND.LOCAL
```

#### Ticket 3 — Cliente sin DNS (`ubu-ventas-01`)

- **Ruptura (solo en runtime, el archivo `/etc/netplan/60-ad-dns.yaml` no se toca).** Se usó `resolvectl` y no `wg-quick down` porque bajar el túnel cortaría el SSH a los clones:

```bash
ssh -i AD.pem ubuntu@10.0.1.12 'IF=$(ip -o -4 route show to default | awk "{print \$5}" | head -1); sudo resolvectl dns $IF 10.0.0.2; echo "IF=$IF"; dig +short +time=3 +tries=1 SRV _ldap._tcp.corp.wonderland.local; echo "dig rc=$?"; realm discover corp.wonderland.local 2>&1 | head -3'
```

- **Síntoma:** interfaz `ens5`; `dig +short SRV` devuelve vacío.
- **Diagnóstico (con caché limpia):**

```bash
ssh -i AD.pem ubuntu@10.0.1.12 'sudo resolvectl flush-caches; resolvectl status ens5 | grep -E "DNS Servers|DNS Domain"; getent hosts win-dc01.corp.wonderland.local; echo "getent rc=$?"; realm discover corp.wonderland.local 2>&1 | head -3'
```

```
DNS Servers: 10.0.0.2
DNS Domain: corp.wonderland.local
getent rc=2
```

> [!warning] `realm discover` no sirve como indicador de DNS
> Siguió respondiendo con el DNS roto (usa la configuración local de sssd/krb5). Para diagnosticar DNS de dominio usar `getent hosts <dc>` y `dig SRV`.

- **Resolución y verificación:**

```bash
ssh -i AD.pem ubuntu@10.0.1.12 'sudo resolvectl dns ens5 192.168.50.10; sudo resolvectl flush-caches; resolvectl status ens5 | grep "DNS Servers"; getent hosts win-dc01.corp.wonderland.local; dig +short SRV _ldap._tcp.corp.wonderland.local; sudo realm list | head -4'
```

```
DNS Servers: 192.168.50.10
192.168.50.10   win-dc01.corp.wonderland.local
0 100 389 win-dc01.corp.wonderland.local.
corp.wonderland.local
  type: kerberos
  realm-name: CORP.WONDERLAND.LOCAL
  domain-name: corp.wonderland.local
```

#### Ticket 4 — Alta de usuario con acceso al share (`lgomez`, Ventas)

- **Resolución (DC):**

```powershell
$p = ConvertTo-SecureString 'Wonderland#2026!Lab' -AsPlainText -Force
New-ADUser -Name "Laura Gomez" -SamAccountName lgomez -UserPrincipalName lgomez@corp.wonderland.local -Path "OU=Ventas,DC=corp,DC=wonderland,DC=local" -AccountPassword $p -Enabled $true -ChangePasswordAtLogon $false
Add-ADGroupMember GG_Ventas -Members lgomez
Get-ADUser lgomez -Properties MemberOf,Enabled | Select SamAccountName,Enabled,MemberOf
```

> [!bug] Error de sintaxis frecuente
> `Add-ADGroupMember GG_Ventas -Member lgomez` falla con `parameter name 'Member' is ambiguous` (colisiona con `-MemberTimeToLive`). Usar **`-Members`**.

```
SamAccountName Enabled MemberOf
lgomez         True    {CN=GG_Ventas,OU=Ventas,DC=corp,DC=wonderland,DC=local}
```

- **Verificación (host, a `ubu-ventas-01`):**

```bash
ssh -i AD.pem ubuntu@10.0.1.12 'sudo sssctl user-checks lgomez@corp.wonderland.local -s sshd -a acct 2>&1 | grep "pam_acct_mgmt:"; smbclient //localhost/ventas -U "CORP\\lgomez%Wonderland#2026!Lab" -c "ls" 2>&1 | grep -E "blocks|NT_STATUS"'
```

```
pam_acct_mgmt: Success
7034376 blocks of size 1024. 4416816 blocks available
```

#### Ticket 5 — Acceso denegado por política de departamento (`lgomez`)

- **Ruptura (DC):**

```powershell
Remove-ADGroupMember GG_Ventas -Members lgomez -Confirm:$false
Get-ADGroupMember GG_Ventas | Select SamAccountName
```

```
SamAccountName
msoto
```

- **Síntoma (host; `sss_cache -E` fuerza el refresco de sssd):**

```bash
ssh -i AD.pem ubuntu@10.0.1.12 'sudo sss_cache -E; sudo sssctl user-checks lgomez@corp.wonderland.local -s sshd -a acct 2>&1 | grep "pam_acct_mgmt:"; smbclient //localhost/ventas -U "CORP\\lgomez%Wonderland#2026!Lab" -c "ls" 2>&1 | grep -E "blocks|NT_STATUS"'
```

```
pam_acct_mgmt: Permission denied
tree connect failed: NT_STATUS_ACCESS_DENIED
```

- **Resolución (DC):** `Add-ADGroupMember GG_Ventas -Members lgomez`; repetir el comando del host:

```
pam_acct_mgmt: Success
7034376 blocks of size 1024. 4394192 blocks available
```

> [!note] Estado en que queda `lgomez`
> El usuario `lgomez` (Ventas, contraseña del lab) permanece en el dominio y en `GG_Ventas`. `msoto` quedó con `Nueva#2026!Lab`.

### F.2 Checklist final del Nivel 1 (resultados)

- **Equipos en su OU (DC):**

```powershell
Get-ADComputer -Filter * -Properties DistinguishedName | Select Name,DistinguishedName
```

```
WIN-DC01        CN=WIN-DC01,OU=Domain Controllers,DC=corp,DC=wonderland,DC=local
UBU-IT-01       CN=UBU-IT-01,OU=IT,DC=corp,DC=wonderland,DC=local
UBU-VENTAS-01   CN=UBU-VENTAS-01,OU=Ventas,DC=corp,DC=wonderland,DC=local
UBU-RRHH-01     CN=UBU-RRHH-01,OU=RRHH,DC=corp,DC=wonderland,DC=local
UBU-GERENCIA-01 CN=UBU-GERENCIA-01,OU=Gerencia,DC=corp,DC=wonderland,DC=local
```

- **`dcdiag /q`:** sin salida (sin errores).
- **Túnel (host):** `sudo wg show wg-hybrid; ping -c2 10.0.1.11`

```
peer: <CLAVE-PUBLICA-SERVER>
  endpoint: <EIP-GATEWAY>:51820
  allowed ips: 10.91.0.0/24, 10.0.0.0/16
  latest handshake: 1 minute, 25 seconds ago
  transfer: 2.55 MiB received, 3.24 MiB sent
  persistent keepalive: every 25 seconds
64 bytes from 10.0.1.11: icmp_seq=1 ttl=63 time=187 ms
2 packets transmitted, 2 received, 0% packet loss
```

- **Matriz de acceso, Samba por departamento, tráfico simulado y 5 tickets:** hechos (E.1, E.3, F.1).

### F.3 Ajustes de cierre

#### Hook de libvirt persistente (resuelve el error 21)

- **Causa:** el hook original solo insertaba las reglas en el `started` de su red y con un guard `grep` que no reinsertaba si la regla ya existía; al arrancar `pentest-lab` después, libvirt vuelve a insertar sus `reject` por encima de los `accept`.
- **Parche** (backup en `/etc/libvirt/hooks/network.bak2`): ante el `started` de `pentest-lab`, `lab-hybrid` o `ad-lab`, borra por handle las reglas previas de cada par bridge/CIDR y las reinserta arriba. Se conservan todas las reglas anteriores y se añaden las de los otros bridges:

```bash
#!/bin/bash
NETWORK="$1"
OPERATION="$2"

add_rule_once() {
    local bridge="$1"
    local cidr="$2"
    for h in $(nft -a list chain ip libvirt_network guest_input 2>/dev/null | grep "oifname \"$bridge\" ip saddr $cidr accept" | awk '{print $NF}'); do
        nft delete rule ip libvirt_network guest_input handle "$h"
    done
    nft insert rule ip libvirt_network guest_input oifname "$bridge" ip saddr "$cidr" accept
}

if [[ "$OPERATION" == "started" ]]; then
    case "$NETWORK" in
        pentest-lab|lab-hybrid|ad-lab)
            add_rule_once "virbr-pentest" "10.90.0.0/24"
            add_rule_once "virbr-pentest" "10.92.0.0/24"
            add_rule_once "virbr-pentest" "10.80.0.0/16"
            add_rule_once "virbr-lab" "10.90.0.0/24"
            add_rule_once "virbr-lab" "10.92.0.0/24"
            add_rule_once "virbr-lab" "10.91.0.0/24"
            add_rule_once "virbr-lab" "10.0.0.0/16"
            ;;
    esac
fi
```

- **Aplicación y verificación** (el archivo no es legible sin `sudo`: `bash -n` también va con `sudo`):

```bash
sudo bash -n /etc/libvirt/hooks/network && sudo /etc/libvirt/hooks/network lab-hybrid started begin - && sudo nft list chain ip libvirt_network guest_input | head -20
```

```
oifname "virbr-lab" ip saddr 10.0.0.0/16 accept
oifname "virbr-lab" ip saddr 10.91.0.0/24 accept
oifname "virbr-lab" ip saddr 10.92.0.0/24 accept
oifname "virbr-lab" ip saddr 10.90.0.0/24 accept
oifname "virbr-pentest" ip saddr 10.80.0.0/16 accept
oifname "virbr-pentest" ip saddr 10.92.0.0/24 accept
oifname "virbr-pentest" ip saddr 10.90.0.0/24 accept
oif "virbr-lab" ip daddr 192.168.50.0/24 ct state established,related ... accept
oif "virbr-lab" ... reject
oif "virbr-pentest" ip daddr 10.10.13.0/24 ct state established,related ... accept
oif "virbr-pentest" ... reject
```

Todos los `accept` quedan sobre los `reject`, sin duplicados.

#### Política de contraseñas (resuelve el error 17)

Estado previo: `MinPasswordLength 7`, `ComplexityEnabled True`, `MaxPasswordAge 42 días`, `LockoutThreshold 5`. GPO existentes (todas `AllSettingsEnabled`): `Restriccion-PanelControl-Ventas-RRHH`, `Default Domain Policy`, `Default Domain Controllers Policy`, `Fondo-Escritorio-Corp`, `Politica-Contrasenas-Corp`. La política de contraseñas de dominio solo la fija la Default Domain Policy (o una PSO): la GPO enlazada a OU no aplica.

```powershell
Set-ADDefaultDomainPasswordPolicy -Identity corp.wonderland.local -MinPasswordLength 8
Get-ADDefaultDomainPasswordPolicy | Select MinPasswordLength,ComplexityEnabled,LockoutThreshold
```

Resultado: `MinPasswordLength 8`, `ComplexityEnabled True`, `LockoutThreshold 5`. Las contraseñas existentes no se ven afectadas.

#### Reloj y RAM del DC

```bash
sudo virsh dumpxml win-dc01 | grep -E "<clock|<memory|<currentMemory|<vcpu"
free -h
```

```
<memory unit='KiB'>6291456</memory>   (6 GB)
<vcpu placement='static'>4</vcpu>
<clock offset='localtime'>
Mem: 15Gi total, 12Gi used, 3.2Gi available · Swap: 8.0Gi, 5.3Gi usado
```

- Reloj: `localtime` confirmado.
- RAM: **no se sube a 9 GB ahora** (+3 GB dejaría al host al límite con swap saturado). 6 GB alcanzan para el Nivel 1; se sube en el Nivel 2 con otras VMs/aplicaciones cerradas.

#### Permisos del USB persistentes (fstab)

El problema real eran `/run/media` y `/run/media/diego`, que udisks recrea con permisos cerrados en cada montaje. Una entrada en fstab monta el disco en la misma ruta que usa el XML de la VM (sin tocarlo) y crea esos directorios con permisos normales; `nofail` evita colgar el arranque si el disco no está:

```bash
sudo cp /etc/fstab /etc/fstab.bak && echo 'UUID=55970ec7-d6df-4660-8d6d-f423930f06dc /run/media/diego/LABDISK ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab && sudo systemctl daemon-reload && sudo findmnt --verify 2>&1 | tail -5; findmnt /run/media/diego/LABDISK
```

`findmnt --verify`: 0 errores de parseo y 3 avisos de la línea previa `/mnt/nextcloud` (sin relación). El montaje actual seguía siendo el de udisks: **la entrada fstab toma efecto en el próximo arranque** y se valida iniciando `win-dc01` sin tocar permisos. Si falla, el plan B son los `chmod o+x` del Anexo D.

#### Limpieza y seguridad de archivos

- AWS: sin limpieza manual (AMI, snapshots y stack los elimina el sandbox).
- `AD.pem` con permisos `600`; la carpeta del proyecto no es un repositorio git. `AD.pem` y `ad-lab.pcap` (contiene credenciales de prueba en NTLM/LDAP simple) se borran y no se suben. Si se inicializa un repositorio: `*.pem` y `*.pcap` al `.gitignore` antes del primer commit.
- Apagado: `sudo wg-quick down wg-hybrid; sudo virsh shutdown win-dc01` (ambos confirmados).

### F.4 Pendientes tras el Nivel 1

- Probar el `stty -echo` del template v3 (segundo prompt de contraseña visible, error 22) en la próxima reconstrucción.
- Validar la entrada fstab del USB en el próximo arranque del host.
- `wg-quick@wg-hybrid` sigue sin habilitarse como servicio (decisión de la sección 11).
- Nivel 2: Route 53 Resolver y DNS híbrido; VPC Endpoints para SSM; subir la RAM del DC.
- Monitoreo v2; imagen real del diagrama de infraestructura; publicación (README/LinkedIn).

### F.5 Filas para la Referencia Operativa Maestra (sección 11)

| Qué | Valor |
|---|---|
| Stack vigente | `ActiveDirectoryLab` (template v3, `DeployClients`); `Active-Directory-Enterprise` fue el del 08-10 |
| Acceso a clones | SSH `ubuntu@10.0.1.x` con `AD.pem` desde el host, por el túnel (SG: 22 desde `10.91.0.0/24`) |
| Samba | `winbind` + sssd, `smb.conf` con `security = ads`, idmap `rid`, `valid users = @"CORP\GG_<dept>"` (E.3, E.5) |
| Hook libvirt | Reinserta los accept de `virbr-lab` y `virbr-pentest` al arrancar `pentest-lab` o `lab-hybrid` (F.3); backups `network.bak`, `network.bak2` |
| Auditoría del DC | `auditpol /set /subcategory:"Credential Validation" /success:enable /failure:enable` |
| USB | Entrada fstab por UUID de `LABDISK` en `/run/media/diego/LABDISK` |
| Errores | 18 a 30 (sección 10); el 17 y el 21 quedaron resueltos |
