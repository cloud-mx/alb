# AWS Application Load Balancer (ALB) - Plantilla Principal

- **Tópico**: `templates/loadbalancing/elb-alb-main.cf-j2.yml`

## Propósito y Resumen

Esta plantilla de AWS CloudFormation, potenciada con Jinja2, despliega un **Application Load Balancer (ALB)** de AWS altamente configurable y seguro. Es el componente central para cualquier arquitectura de aplicación web y está diseñada para cumplir con los Estándares de Seguridad Fundacional en la Nube de GE Vernova (GEV FCSC).

La plantilla crea el balanceador de carga, un grupo de seguridad dedicado para él, y opcionalmente, un listener de redirección de HTTP a HTTPS. **No crea listeners principales (como el del puerto 443) ni Target Groups**, ya que estos deben ser desplegados con plantillas modulares separadas para promover la reutilización.

## Alcance

- **Dentro del Alcance:**
    - Creación del recurso `AWS::ElasticLoadBalancingV2::LoadBalancer`.
    - Creación de un `AWS::EC2::SecurityGroup` exclusivo para el ALB.
    - Asociación obligatoria con un **AWS WAF** si el esquema es `internet-facing`.
    - Configuración de registros de acceso (`access logs`).
    - Creación condicional de un listener en el puerto 80 para redirigir a HTTPS.
- **Fuera del Alcance:**
    - Creación de la VPC, Subnets, WAF WebACL, Bucket S3 para logs, y Certificados ACM. Estos son prerrequisitos.
    - Creación de Listeners principales (ej. HTTPS) y Target Groups. Utilice las plantillas modulares correspondientes.

## Prerrequisitos

Antes de desplegar esta plantilla, asegúrese de tener:
1.  **VPC ID y Subnet IDs:** Una VPC existente y al menos dos subredes (preferiblemente privadas) donde se desplegará el ALB.
2.  **ARN de WAF Web ACL (si es público):** El ARN de una `WebACL` de AWS WAF v2 que se asociará al ALB si es `internet-facing`.
3.  **Nombre de Bucket S3 para Logs:** Un bucket S3 existente donde se almacenarán los registros de acceso del ALB.

## Parámetros

### Parámetros de Jinja2 (`jinjaparams`)

Estos parámetros se definen en el archivo de entrada y controlan la lógica de renderizado de la plantilla.

| Nombre | Tipo | Descripción | Ejemplo | Opcional? |
| :--- | :--- | :--- | :--- | :--- |
| `idle_timeout_seconds` | Number | Tiempo en segundos que el ALB espera antes de cerrar una conexión inactiva. | `60` | Sí (defecto: `60`) |
| `drop_invalid_headers` | Boolean | Si es `true`, el ALB descarta cabeceras HTTP inválidas. | `true` | Sí (defecto: `true`) |
| `create_http_redirect` | Boolean | Si es `true`, crea un listener en el puerto 80 que redirige todo el tráfico a HTTPS (puerto 443). | `true` | Sí (defecto: `true`) |
| `extra_tags` | List | Lista de etiquetas adicionales para aplicar a los recursos. | `[{ "Key": "CostCenter", "Value": "12345" }]` | Sí |

### Parámetros de CloudFormation

Estos son los parámetros estándar que se pasan al stack de CloudFormation.

| Nombre | Tipo | Descripción | Restricciones | Opcional? |
| :--- | :--- | :--- | :--- | :--- |
| `UAI` | String | El identificador único de la aplicación. | `^uai[0-9]{7}$` | **No** |
| `AppName` | String | Nombre corto de la aplicación. | `^[a-z][a-z0-9-]*`, 3-20 chars | **No** |
| `Env` | String | Entorno de despliegue. | `dev`, `qa`, `stg`, `prd`, `lab` | **No** |
| `Scheme` | String | Define si el ALB es interno o público. | `internal`, `internet-facing` | **No** |
| `SubnetIds` | `List<AWS::EC2::Subnet::Id>` | Lista de Subnet IDs donde desplegar el ALB. | - | **No** |
| `WebAclArn` | String | **REQUERIDO si `Scheme` es `internet-facing`**. ARN de la Web ACL de WAF. | - | Sí (pero forzado por condición) |
| `AccessLogsS3BucketName` | String | Nombre del bucket S3 para los registros de acceso. | - | **No** |

## Salidas (Outputs)

| Nombre | Descripción |
| :--- | :--- |
| `LoadBalancerArn` | ARN del Application Load Balancer creado. |
| `LoadBalancerDNSName` | Nombre DNS público o interno del balanceador de carga. |
| `CanonicalHostedZoneId` | ID de la zona alojada canónica para el alias de Route 53. |
| `LoadBalancerSecurityGroupId` | ID del grupo de seguridad creado para el balanceador de carga. |

## Ejemplo de Uso (`stack_input_prd.yml`)

```yaml
stacks:
  us1:
    MiAppALB:
      template: templates/loadbalancing/elb-alb-main.cf-j2.yml
      params:
        UAI: "uai1234567"
        AppName: "miapi"
        Env: "prd"
        Scheme: "internet-facing"
        SubnetIds: ["subnet-0123abc...", "subnet-0456def..."]
        WebAclArn: "arn:aws:wafv2:us-east-1:111122223333:regional/webacl/GEV-Standard-Prod/..."
        AccessLogsS3BucketName: "gevernova-central-logs"
      jinjaparams:
        idle_timeout_seconds: 300
        create_http_redirect: true
        extra_tags:
          - Key: "DataClassification"
            Value: "Confidential"
