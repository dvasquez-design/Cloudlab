# CloudLab

Un laboratorio híbrido (on-premise + AWS) donde aprendo cloud, networking y seguridad construyendo, rompiendo, investigando y documentando entornos reales.

> 🚧 **Trabajo en progreso.** Este README describe los objetivos y la estructura del proyecto. El contenido se va publicando entorno por entorno.

## Qué es

CloudLab es un laboratorio personal construido alrededor de una idea: máquinas virtuales corriendo en mi propio hardware (KVM) conectadas de forma segura a infraestructura en AWS, para practicar cómo funcionan los entornos híbridos reales.

Cada caso de uso es un **entorno independiente** en su propio subdirectorio, con su propia documentación, para que el laboratorio pueda crecer sin convertirse en un solo proyecto enredado.

## Objetivos

- **Aprender haciendo.** Entender cómo funcionan sistemas más grandes empezando de a poco: primero una red, luego un servicio, luego cómo se conectan y cómo se monitorean.
- **Conectividad híbrida.** Conectar VMs on-premise con una VPC de AWS a través de un túnel WireGuard iniciado desde el lado local, sin exponer puertos entrantes.
- **Infraestructura como código.** Definir el lado de AWS con CloudFormation para que cada entorno pueda desplegarse, destruirse y volver a desplegarse de forma reproducible.
- **Seguridad desde ambos lados.** Practicar red team y blue team en un entorno controlado, con monitoreo, detección y un reporte escrito para cada ejercicio.
- **Documentar todo.** Cada entorno registra no solo lo que funcionó, sino las decisiones, los problemas encontrados y cómo se resolvieron.
- **Compartir plantillas reutilizables.** Publicar las plantillas para que otros puedan replicar los entornos, con un portfolio web estático planeado para presentarlos.

## Arquitectura (vista general)

```
  RED LOCAL (on-premise)                              AWS
 +---------------------------+              +---------------------------+
 |  Host KVM                 |              |  VPC                      |
 |   +-------+  +-------+    |   Túnel      |   +--------------------+  |
 |   | VM    |  | VM    |    |   WireGuard  |   | Herramientas /     |  |
 |   +-------+  +-------+    |<============>|   | monitoreo          |  |
 |                           |  iniciado    |   +--------------------+  |
 |  (sin puertos entrantes   |  desde local |   desplegado con           |
 |   abiertos)               |              |   CloudFormation          |
 +---------------------------+              +---------------------------+
```

## Entornos

| Entorno | Estado | Descripción |
|---|---|---|
| [`PentestLab/`](./PentestLab) | 🟡 En progreso | Análisis de VMs vulnerables como red team y blue team, con un stack de monitoreo en AWS conectado a las VMs on-premise. |
| [`Active-Directory-Hibrido/`](./Active-Directory-Hibrido) | 🟡 En progreso (Nivel 1 completo) | Dominio de Active Directory on-premise (Windows Server 2022 en KVM) extendido a AWS por WireGuard: clientes Ubuntu unidos al dominio por departamento, shares Samba, tráfico simulado y tickets de soporte N1. |
| Más entornos | 🔜 Planeados | Se irán agregando como subdirectorios independientes a medida que estén listos. |

### Estado de PentestLab

- ✅ Mr. Robot: análisis completo publicado
- 🔄 Análisis de vulnerabilidades con Nessus
- 🔄 Reporte final
- 🔄 Enfoque purple-team: cruzar lo que ve el ataque con lo que detecta el monitoreo

### Estado de Active-Directory-Hibrido

**Nivel 1 (completo)**

- ✅ Controlador de dominio `corp.wonderland.local` on-premise, con OUs, usuarios y grupos por departamento
- ✅ Túnel WireGuard entre el host KVM y un gateway en AWS, desplegado con CloudFormation
- ✅ 4 clientes Ubuntu en AWS unidos al dominio, uno por departamento (IT, Ventas, RRHH, Gerencia), con acceso controlado por grupo
- ✅ Shares Samba por departamento con rechazo cruzado verificado
- ✅ Tráfico de autenticación simulado y capturado
- ✅ 5 tickets de soporte N1 resueltos y documentados

**Próxima versión (Nivel 2, planificado)**

- 🔜 DNS híbrido nativo con Route 53 Resolver, sin configurar el DNS a mano en cada cliente
- 🔜 Políticas de contraseña diferenciadas por OU (Fine-Grained Password Policies)
- 🔜 LAPS: contraseña de administrador local rotativa
- 🔜 Hardening: SMBv1 deshabilitado y auditoría de NTLM
- 🔜 Auditoría avanzada de eventos de Windows y VPC Flow Logs
- 🔜 Escenarios de troubleshooting N2/N3 con evidencia
- 🔜 Monitoreo del entorno (fuera del alcance del Nivel 1)

## Estructura del repositorio

```
Cloudlab/
├── README.md
├── PentestLab/
└── Active-Directory-Hibrido/
```

Se irán agregando más subdirectorios con cada nuevo entorno.

## Principios

- Todo se despliega en un entorno controlado que me pertenece.
- Cada ejercicio se documenta con su razonamiento, no solo con su resultado.
- Nada se publica como terminado hasta que lo esté.

## Autor

Diego Vásquez · [GitHub](https://github.com/dvasquez-design) · [LinkedIn](https://linkedin.com/in/diego-vasquez-cloud)
