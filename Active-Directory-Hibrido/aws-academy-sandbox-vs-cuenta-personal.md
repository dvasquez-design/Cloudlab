# Active Directory Híbrido fuera del sandbox: AWS Academy Learner Lab vs. cuenta AWS normal

El diseño de `hybrid-ad-lab-network.yaml` tiene las mismas decisiones heredadas del sandbox que ya se documentaron para [PentestLab](../PentestLab/aws-academy-sandbox-vs-cuenta-personal.md) (mismo AWS Academy Learner Lab, mismo bloqueo de `iam:PassRole`). Este documento aplica ese mismo análisis a este subproyecto específicamente — arquitectura de dos subredes + gateway con doble ENI + 4 clientes, en vez del Attacker Node único de PentestLab — sin modificar el template.

---

## 1. Qué es distinto entre un Learner Lab y una cuenta normal

La tabla es la misma que en PentestLab (es el mismo sandbox): rol/instance profile pre-creados sin `iam:PassRole` disponible para CloudFormation, presupuesto fijo con auto-terminación, sesiones con límite de horas. Ver el documento de PentestLab para el detalle completo — acá solo lo específico de este subproyecto.

## 2. Dónde aparece la restricción en este template

A diferencia de PentestLab (una sola instancia), acá el `iam:PassRole` bloqueado afecta **5 instancias EC2**: el gateway (`GatewayInstance`) y los 4 clientes (`ClientIt`, `ClientVentas`, `ClientRrhh`, `ClientGerencia`). El parámetro `ExistingInstanceProfileName` es compartido por las cinco (`IamInstanceProfile: !If [HasInstanceProfile, !Ref ExistingInstanceProfileName, !Ref "AWS::NoValue"]` en cada recurso), así que en Academy el checklist post-deploy se repite 5 veces: asignar `LabInstanceProfile` desde consola (EC2 > Actions > Security > Modify IAM role) y reiniciar cada instancia — documentado en el Anexo E del documento completo del lab.

**¿Hace falta el rol acá?** A diferencia de PentestLab, acá **sí** — al menos para el gateway. `GatewayWanSecurityGroup` solo abre `UDP 51820` (WireGuard); no tiene ninguna regla de entrada para el puerto 22, ni siquiera restringida a una IP. Eso significa que antes de que el túnel esté arriba, SSM es el **único** canal para entrar al gateway — para diagnosticar un `UserData` que falló, leer `server_public.key` en el primer boot, o revisar `/var/log/user-data.log`. Sin el rol, un gateway que no levanta el túnel por sí solo queda inaccesible salvo que se destruya y recree el stack.

Para los 4 clientes la situación es la de PentestLab: `PrivateSecurityGroup` sí permite el puerto 22, pero solo desde `TunnelCidr` (`10.91.0.0/24`) — una vez que el túnel está arriba, se puede entrar por SSH a través de él sin depender de SSM. El rol IAM ahí es un canal alterno, no el único; en el gateway, mientras el túnel no exista, es el único.

## 3. Resolverlo en una cuenta normal

Mismo patrón que PentestLab: crear el rol una vez, vincularlo a un instance profile, y pasar su nombre como parámetro — sin tocar el template.

```bash
# 1. Rol con trust policy para EC2 (una sola vez, reusable para las 5 instancias)
aws iam create-role \
  --role-name ad-hybrid-ec2-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

# 2. Policy gestionada de AWS para SSM
aws iam attach-role-policy \
  --role-name ad-hybrid-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# 3. Instance profile
aws iam create-instance-profile --instance-profile-name ad-hybrid-ec2-profile
aws iam add-role-to-instance-profile \
  --instance-profile-name ad-hybrid-ec2-profile \
  --role-name ad-hybrid-ec2-role
```

Deploy en dos pasos, igual que el flujo documentado para Academy (gateway primero, clientes después), pero sin el checklist manual:

```bash
# Paso 1 — red + gateway, sin clientes
aws cloudformation deploy \
  --template-file hybrid-ad-lab-network.yaml \
  --stack-name ad-hybrid-lab \
  --parameter-overrides \
    WgClientPublicKey=<clave-publica-de-wonderland> \
    KeyName=AD \
    ExistingInstanceProfileName=ad-hybrid-ec2-profile \
    DeployClients=false \
  --capabilities CAPABILITY_IAM

# Verificar el túnel y el DC (igual que en Academy: wg-hybrid arriba, win-dc01 encendido)

# Paso 2 — update del mismo stack para lanzar los 4 clientes
aws cloudformation deploy \
  --template-file hybrid-ad-lab-network.yaml \
  --stack-name ad-hybrid-lab \
  --parameter-overrides \
    WgClientPublicKey=<clave-publica-de-wonderland> \
    KeyName=AD \
    ExistingInstanceProfileName=ad-hybrid-ec2-profile \
    DeployClients=true \
  --capabilities CAPABILITY_IAM
```

Con `ExistingInstanceProfileName` ya resuelto en el propio deploy, el rol queda asignado a las 5 instancias desde el arranque — sin el paso manual repetido 5 veces ni el reboot posterior por cada una.

## 4. Lo que el sandbox hacía por vos (y en cuenta normal pasa a ser tu responsabilidad)

Mismas responsabilidades que en PentestLab (apagado automático, tope de gasto, destruir el stack al cerrar sesión) — ver sección 3 del documento de PentestLab para los comandos de `AWS Budgets`. Una particularidad de este subproyecto: `KeyName` tiene default `AD` (a diferencia de PentestLab, no se resetea entre sesiones en una cuenta normal, pero igual conviene confirmar que el key pair exista en la región antes de desplegar).

## 5. Costos estimados (cuenta normal, fuera de free tier)

Diferencia clave respecto a PentestLab: acá son **5 instancias `t3.micro`** (no 2× `t3.medium`), más **1 sola IPv4 pública** (la EIP del gateway — los 4 clientes están en la subred privada, sin IP pública) y **1 EIP explícita** (a diferencia de PentestLab, que usa la IP automática de `MapPublicIpOnLaunch`; acá `GatewayWanEip` es un recurso `AWS::EC2::EIP` fijo mientras el stack exista).

Precios de referencia en `us-east-1` ([EC2 t3.micro on-demand](https://www.doit.com/compute/spot/us-east-1/t3.micro), [EBS gp3](https://cloudoptimo.com/blog/everything-you-need-to-know-about-aws-ebs-pricing-and-cost-saving-tips/), [IPv4 pública](https://aws.amazon.com/vpc/pricing/), [transferencia de datos](https://egresscost.com/aws/data-transfer-pricing)):

| Recurso | Precio unitario | Costo si corre 24/7 (730h/mes) |
|---|---|---|
| 5× EC2 `t3.micro` (gateway + 4 clientes) | $0.0104/hr c/u | ~$37.96/mes |
| 1× EIP (gateway, asociada) | $0.005/hr | ~$3.65/mes |
| 5× EBS gp3, tamaño default de la AMI (sin `BlockDeviceMappings` en el template — ~8GB c/u, 40GB total) | $0.08/GB-mes | ~$3.20/mes |
| Transferencia saliente | 100GB/mes gratis — el tráfico real de este lab (Kerberos, LDAP, SMB, DNS entre on-premise y AWS) es de bajo volumen | ~$0 en uso normal |
| **Total corriendo 24/7** | | **~$44.81/mes** |

Con uso real tipo "sesión de fin de semana, ~20h al mes" (que es como se usó efectivamente, según el estado del lab — DC y túnel apagados entre sesiones):

| Recurso | Costo en ~20h de uso |
|---|---|
| 5× EC2 `t3.micro` | ~$1.04 |
| 1× EIP (mientras el stack existe) | ~$0.10 |
| EBS (prorrateado, si se destruye el mismo día) | ~$0.09 |
| **Total aprox. por sesión** | **~$1.23** |

**Nota sobre la EIP:** a diferencia de PentestLab (IP pública automática, sin costo cuando la instancia corre — solo se cobra la hora de IPv4 en uso), acá `GatewayWanEip` es un recurso de Elastic IP explícito. Una Elastic IP **asociada a una instancia en ejecución** cobra la misma tarifa de IPv4 en uso ($0.005/hr) — el costo adicional real de usar Elastic IP en vez de la automática aparece solo si la instancia se detiene y la EIP queda reservada sin asociar, o si se mantiene una segunda EIP sin usar. Mientras el stack completo se destruye entre sesiones (recomendado, igual que en PentestLab), este caso no aplica.

```bash
# Al cerrar sesión: destruir todo el stack, no solo detener instancias
# (mismo motivo que en PentestLab: EBS y EIP reservada siguen facturando)
aws cloudformation delete-stack --stack-name ad-hybrid-lab
```

## 6. Resumen rápido

- **Sandbox Academy:** sin costo directo, fricción operativa 5x mayor que PentestLab (checklist manual por cada una de las 5 instancias).
- **Cuenta normal, deploy/destroy por sesión:** rol creado una vez, deploy en 2 pasos sin fricción manual, costo marginal (~USD 1.25 por sesión de fin de semana — más barato que PentestLab por usar `t3.micro` en vez de `t3.medium`).
- **Cuenta normal, stack dejado corriendo:** ~USD 45/mes — bastante menos que PentestLab (~USD 73/mes) por el mismo motivo de instance type, pero sigue siendo el error más caro posible si se olvida apagarlo.
