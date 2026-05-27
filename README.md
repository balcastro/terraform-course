# Curso de Terraform en AWS — Temario completo

---

## Módulo 0 — Entorno y fundamentos

**Conceptos:** Qué es IaC y por qué Terraform, ciclo `init/plan/apply/destroy`, estructura de ficheros de un proyecto, qué es un provider y cómo se versiona.

El repositorio Git como fuente de verdad del código: `.gitignore` para Terraform (`.tfstate`, `.tfstate.backup`, `.terraform/`, `*.tfvars` con secretos), convención de commits para cambios de infraestructura.

`terraform fmt` — formateador oficial. Estandariza indentación y estilo. Sin él el código diverge entre personas del equipo. Se integra como pre-commit hook para que nunca llegue código sin formatear al repositorio.

`terraform validate` — valida la sintaxis y coherencia interna del código sin conectarse a AWS. No detecta todos los errores pero sí los más básicos. Se ejecuta antes del plan en cualquier pipeline serio.

`.terraform.lock.hcl` — fichero de lock de versiones de providers generado por `terraform init`. Fija las versiones exactas (con hashes) para que todos los miembros del equipo y el CI usen exactamente el mismo binario de provider. Va a Git. `terraform providers lock` para actualizar de forma controlada.

`terraform plan -out=planfile` + `terraform apply planfile` — flujo correcto en entornos reales: el plan se genera, se revisa (o se adjunta a un PR), y el apply ejecuta exactamente ese plan ya aprobado, no uno nuevo generado en el momento. Evita que el estado de AWS cambie entre el plan y el apply.

**Ejercicio:** Instalar Terraform y AWS CLI. Configurar credenciales con `aws configure`. Inicializar repo Git, crear `.gitignore` correcto para Terraform. Escribir el primer `main.tf` con el provider de AWS. `terraform init` — observar la creación de `.terraform.lock.hcl`. `terraform fmt` y `terraform validate`. Primer commit incluyendo el lock file.

---

## Módulo 1 — Primer recurso real y el state

**Conceptos:** Bloque `resource`, el fichero `.tfstate` (qué contiene, por qué es la fuente de verdad, por qué nunca va a Git), `terraform show`, `terraform state list`, ciclo de vida básico de un recurso.

**Ejercicio:** Crear un bucket S3. Explorar el state con `terraform show` y `terraform state list`. Modificar un atributo y observar el diff en el plan. Destruirlo. Verificar que el state no ha acabado en el repositorio.

---

## Módulo 2 — Variables y outputs

**Conceptos:** `variable` con tipos (`string`, `number`, `bool`, `list`, `map`, `object`), valores por defecto, `terraform.tfvars` (cuáles van a Git y cuáles no), `-var`, `TF_VAR_*`. Bloque `output`.

**Ejercicio:** Parametrizar el bucket del módulo anterior. Añadir outputs. Practicar las distintas formas de pasar valores. Commit con los `.tfvars` que no contienen secretos.

---

## Módulo 3 — Locals, datasources y tags

**Conceptos — Locals y datasources:** `locals` para centralizar lógica y evitar repetición. `data sources` para leer de AWS: cuenta actual, región, AMIs, zonas de disponibilidad. Referencias entre recursos con `resource.tipo.nombre.atributo`.

**Conceptos — Tags y common_tags:** AWS permite taggear casi cualquier recurso. Los tags son la principal herramienta para gestión de costes, auditoría, búsqueda y políticas de acceso (IAM conditions sobre tags). Buenas prácticas mínimas: `Environment`, `Project`, `ManagedBy`, `Owner`.

El patrón `common_tags` consiste en definir un mapa de tags base en locals y aplicarlo en el bloque `default_tags` del provider para que se hereden automáticamente a todos los recursos:

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.env
      Project     = var.project
      ManagedBy   = "terraform"
      Owner       = "infra-team"
    }
  }
}
```

Con `default_tags` todos los recursos reciben esos tags sin declararlo en cada uno. Tags específicos de un recurso se añaden en su bloque `tags` y se fusionan automáticamente. Si hay conflicto de clave, el tag del recurso sobreescribe el del provider.

Cuándo usar `default_tags` vs `merge` en locals: `default_tags` es más limpio y no requiere disciplina por recurso, pero no aparece en el plan de forma explícita. `merge` con locals es más explícito y portable si se usan múltiples providers.

**Ejercicio:** Construir un nombre de recurso dinámico con `locals` usando `data "aws_caller_identity"` y `data "aws_region"`. Configurar `default_tags` en el provider con `Environment`, `Project`, `ManagedBy = "terraform"` y `Owner`. Crear un bucket S3 sin bloque `tags` y verificar en la consola que los tags aparecen. Añadir un tag específico al bucket (`Purpose = "assets"`) y verificar que coexiste con los comunes. Intentar sobreescribir `ManagedBy` desde el recurso y observar el comportamiento.

---

## Módulo 4 — Funciones integradas

**4.1 — Cadenas**

Funciones: `format`, `join`, `split`, `replace`, `trimspace`, `upper`, `lower`, `substr`.

Ejercicio: Construir nombres de recursos normalizados (sin espacios, en minúsculas, con prefijos) a partir de variables de entorno y proyecto. Ej: `format("%s-%s-%s", var.project, var.env, var.region)`.

**4.2 — Colecciones**

Funciones: `toset`, `tolist`, `tomap`, `flatten`, `merge`, `keys`, `values`, `lookup`, `concat`, `length`, `contains`, `distinct`, `zipmap`.

Ejercicio: Limpiar una lista de nombres con posibles duplicados con `distinct` + `toset`. Combinar dos mapas de tags con `merge` (tags comunes + tags específicas del recurso). Construir un mapa `{nombre → cidr}` con `zipmap` a partir de dos listas paralelas.

**4.3 — Red**

Funciones: `cidrsubnet`, `cidrhost`, `cidrnetmask`, `cidrsubnets`.

Ejercicio: Dado `10.0.0.0/16`, calcular automáticamente CIDRs para 4 subnets con `cidrsubnet`. Calcular la IP del gateway (primera usable) con `cidrhost`. Usar `cidrsubnets` para repartir bloques de tamaño distinto en un solo paso.

**4.4 — Condicionales y control de errores**

Funciones: `can`, `try`, operador ternario, `coalesce`, `coalescelist`, `one`.

Ejercicio: Variable opcional de CIDR: si no se pasa, calcularlo con `cidrsubnet`; si se pasa, usarlo directamente. Usar `try` para leer un atributo que puede no existir en un objeto. `coalesce` para resolver un valor con fallback.

**4.5 — Expresiones `for`**

No son funciones pero se usan junto a ellas: transformar listas en mapas, filtrar con `if`, invertir un mapa.

Ejercicio: Convertir una lista de strings en un mapa `{nombre → nombre_upper}`. Filtrar un mapa según un atributo. Invertir un mapa `{az → cidr}` a `{cidr → az}`.

---

## Módulo 5 — Iteradores

**Conceptos:**

`count`: genera recursos indexados numéricamente (`resource[0]`, `resource[1]`). Simple pero frágil: eliminar un elemento del medio reindica todo lo posterior y Terraform recrea los recursos afectados. Solo recomendable para recursos idénticos sin identidad propia.

`for_each` con sets: cada elemento se identifica por su valor en el state (`resource["valor"]`). Eliminar un elemento solo destruye ese. Limitación: elementos deben ser strings, sin orden garantizado.

`for_each` con maps: la clave del mapa es el identificador en el state (`resource["clave"]`). Opción más robusta: naming explícito, múltiples atributos por elemento, añadir o eliminar entradas no afecta al resto. Patrón recomendado por defecto.

`dynamic` blocks: para bloques repetidos dentro de un recurso (ej. reglas de un SG), no para recursos completos.

**Ejercicio A — Demostración de reordenación:** Crear 3 buckets con `count` a partir de una lista. Eliminar el del medio, observar en el plan que Terraform quiere modificar/recrear los siguientes. Repetir con `for_each` sobre un set y ver que solo elimina el bucket correcto.

**Ejercicio B — Mapa como fuente de verdad:** Definir un mapa de subnets `{ "pub-a" = { cidr = "...", az = "..." }, "pub-b" = { ... } }`. Crear las subnets con `for_each`. Añadir una tercera entrada y verificar que el plan solo crea el recurso nuevo sin tocar los existentes.

**Ejercicio C — dynamic blocks:** Security group con reglas definidas como lista de objetos, generando los bloques `ingress` con `dynamic`.

---

## Módulo 6 — Providers avanzados: alias y multi-región

**Conceptos:** Un provider block por defecto configura una región. El alias permite declarar múltiples instancias del mismo provider con configuraciones distintas (región, cuenta, rol asumido). Casos de uso habituales: replicación de recursos en otra región, certificados ACM que deben estar en `us-east-1` (requisito de CloudFront), despliegues multi-cuenta con `assume_role`.

```hcl
provider "aws" {
  region = "eu-west-1"
}

provider "aws" {
  alias  = "us"
  region = "us-east-1"
}
```

Cómo usarlo en un recurso con `provider = aws.us`. Cómo pasarlo a un módulo con el meta-argumento `providers`. Cómo declarar en un módulo que espera un provider con alias mediante `required_providers` y `configuration_aliases`.

**Ejercicio A — Recurso en otra región:** Crear un bucket S3 en `eu-west-1` (provider por defecto) y otro en `us-east-1` (provider con alias). Observar en el state que ambos coexisten y se gestionan de forma independiente.

**Ejercicio B — ACM en us-east-1:** Crear un certificado ACM en `us-east-1` usando el provider con alias, que es el requisito para usarlo con CloudFront. El resto de recursos siguen en la región principal. Discutir por qué este patrón aparece casi siempre en arquitecturas con CDN.

**Ejercicio C — Pasar provider a un módulo:** Extraer la creación del certificado a un módulo. Declarar `configuration_aliases` en el `required_providers` del módulo y pasarle el provider con alias desde el módulo raíz. Verificar que el módulo no asume ninguna región hardcodeada.

---

## Módulo 7 — Red en AWS (VPC)

**Conceptos:** VPC, subnets públicas y privadas, internet gateway, route tables y asociaciones. Qué cuesta dinero (NAT Gateway, Elastic IPs sueltas) y qué no.

**Ejercicio:** VPC con 2 subnets públicas en distintas AZs. CIDRs calculados con `cidrsubnet` (módulo 4). Subnets creadas con `for_each` sobre un mapa (módulo 5). Internet gateway y route table. Sin NAT Gateway.

---

## Módulo 8 — EC2 en capa gratuita

**Conceptos:** Security groups, key pairs, AMIs via `data "aws_ami"`, `user_data`, tipos de instancia free tier (`t2.micro` / `t3.micro`). Bloque `lifecycle`.

**Ejercicio:** EC2 `t2.micro` en la VPC anterior. SG con puertos 22 y 80. `user_data` que instale nginx. Acceder por SSH y verificar. Destruir.

---

## Módulo 9 — Módulos

**Conceptos — Módulos propios:** Qué es un módulo (cualquier directorio con `.tf`). Módulo raíz vs módulos hijos. Cuándo extraer uno: cuando la misma combinación de recursos se instancia más de una vez, o cuando el bloque de responsabilidad es claro. Inputs (`variable`) y outputs del módulo. `source` local. Propagación de outputs entre módulos.

**Conceptos — Módulos del Registry:** El Terraform Registry publica módulos mantenidos por HashiCorp, vendors y comunidad. Un módulo del Registry abstrae decenas de recursos en una interfaz de variables. Ventajas: ahorro de tiempo, patrones probados. Inconvenientes: opacidad, defaults que pueden no ser los esperados, versiones que rompen compatibilidad. Cómo fijar la versión con `version` para evitar sorpresas en el `terraform init`.

**Ejercicio A — Módulo local:** Extraer la VPC a `modules/network` y la EC2 a `modules/compute`. El módulo de red acepta el CIDR base y calcula subnets internamente. Instanciar el módulo de red dos veces con distintos CIDRs. Verificar que los outputs de red llegan correctamente a compute.

**Ejercicio B — Módulo del Registry:** Sustituir el módulo local de red por `terraform-aws-modules/vpc/aws`. Leer la documentación, mapear las variables necesarias, aplicar. Comparar el state resultante con el del ejercicio anterior: observar cuántos recursos crea el módulo por debajo. Fijar la versión. Discutir diferencias en control y legibilidad frente al módulo propio.

---

## Módulo 10 — Organización por entornos

**Conceptos:**

**Opción A — Carpetas por entorno:** `environments/dev/` y `environments/prod/`, cada carpeta con su `main.tf`, `terraform.tfvars` y state propio. Módulos compartidos en `modules/`. Clara separación física, cada entorno se aplica de forma independiente. Riesgo: divergencia de código entre entornos sin disciplina.

**Opción B — Rama Git por entorno:** mismos ficheros en raíz, rama `dev` y rama `prod`, distintos `terraform.tfvars` por rama (o vía CI). El código evoluciona en `dev` y se promueve a `prod` por merge. El historial de Git refleja exactamente lo desplegado en cada entorno. Requiere disciplina de branching y que cada rama apunte a su propio state remoto.

Tradeoffs: carpetas son más simples y explícitas; ramas acoplan el modelo de despliegue al modelo de branching, lo que puede ser una ventaja (trazabilidad) o una complicación (conflictos, cherry-picks).

**Ejercicio:** Implementar la opción A con `dev` y `prod`. Como extensión, diseñar cómo quedaría la estructura de ramas para la opción B y qué cambiaría en el backend config.

---

## Módulo 11 — State remoto

**Conceptos:** Por qué el state local es un problema en equipo. Backend S3 + DynamoDB para locking. `terraform init -migrate-state`. `terraform_remote_state` para leer outputs de otro state. Nunca commitear `.tfstate`.

**Ejercicio A — Migración a backend remoto:** Bucket S3 y tabla DynamoDB para el backend. Migrar el state de `dev` a remoto.

**Ejercicio B — Lectura cruzada de states:** Leer un output del state de `dev` desde `prod` con `terraform_remote_state`. Discutir cuándo usar esto vs duplicar la variable en el `tfvars`.

**Ejercicio C — Subzona DNS delegada entre entornos:** El entorno `prod` gestiona la zona Route53 de `midominio.com`. El entorno `pre` lee el state de `prod` con `terraform_remote_state` para obtener el `zone_id` de la zona padre, crea la zona `pre.midominio.com` y genera los registros NS. De vuelta en `prod`, un data source del state de `pre` lee los nameservers y crea la delegación con un registro NS en la zona padre. Discutir el orden de aplicación (prod primero sin delegación → apply pre → apply prod de nuevo) y por qué este patrón aparece en entornos reales con múltiples cuentas AWS.

---

## Módulo 12 — Importar recursos existentes

**Conceptos:** `terraform import` clásico (solo toca el state, no genera HCL). Bloque `import` de Terraform >= 1.5 que sí genera configuración. Flujo recomendado: generar, revisar, ajustar hasta plan limpio.

**Ejercicio:** Crear un bucket a mano en la consola. Importarlo con el bloque moderno, revisar el HCL generado, ajustar hasta `terraform plan` sin cambios. Commit del resultado.

---

## Módulo 13 — Ciclo de vida y dependencias

**Conceptos:** `lifecycle` blocks: `create_before_destroy`, `prevent_destroy`, `ignore_changes`, `replace_triggered_by`. `depends_on` explícito vs implícito. `terraform apply -replace`. `terraform graph` para visualizar dependencias.

**Ejercicio:** `ignore_changes` en tags de una EC2 simulando cambios manuales que no se quieren revertir. `create_before_destroy` en un security group para reemplazo sin downtime. Forzar recreación de un recurso con `-replace`. Visualizar el grafo del proyecto.

---

## Módulo 14 — Mover y refactorizar el state

**Conceptos:** `terraform state mv` para renombrar o mover recursos a módulos sin destruirlos. `terraform state rm` para desvincular un recurso del state sin destruirlo en AWS. Bloque `moved` (>= 1.1) para documentar refactors en el propio código. Bloque `removed` (>= 1.7). Cuándo cada opción.

**Ejercicio:** Tomar un recurso creado fuera de módulo, moverlo dentro de un módulo con `state mv` + bloque `moved`. Verificar plan limpio. Hacer `state rm` sobre un recurso y observar qué ocurre en el siguiente apply. Commit del bloque `moved` como parte del refactor.

---

## Módulo 15 — Secretos, IAM y gestión de policies

**Conceptos — Datos sensibles:** `sensitive = true` en variables y outputs. Por qué los secretos en el state son un problema estructural (el state no es un secret manager). SSM Parameter Store y Secrets Manager como data sources. Nunca hardcodear credenciales. Qué va a Git y qué no.

**Conceptos — Policies como código:** Las policies de IAM son JSON. Tres formas de gestionarlas en Terraform, de menos a más mantenible:

`jsonencode()`: escribir la policy directamente en HCL como objeto y serializar a JSON. Terraform valida la sintaxis, los valores pueden ser variables o referencias a otros recursos. Recomendado para policies simples o dinámicas.

`file()`: cargar una policy desde un fichero `.json` externo. Útil cuando la policy ya existe o la gestiona otro equipo. Sin interpolación: el fichero se usa tal cual.

`templatefile()`: cargar un fichero `.json.tftpl` con placeholders (`${bucket_arn}`, `${account_id}`) que Terraform sustituye al renderizar. Combina la legibilidad de tener la policy en su propio fichero con la capacidad de parametrizarla.

`data "aws_iam_policy_document"`: datasource nativo que genera el JSON en HCL puro. Ventaja: composición (heredar y sobreescribir statements), validación de campos, diff legible en el plan. Desventaja: aleja la policy de su representación JSON real.

Cuándo cada uno: `jsonencode` para policies cortas y dinámicas; `templatefile` cuando la policy es larga o la gestiona un equipo que prefiere JSON; `aws_iam_policy_document` cuando se necesita composición.

**Ejercicio A — jsonencode:** Crear un bucket S3 y una policy que permita `s3:GetObject` solo desde una IP variable. La policy se construye con `jsonencode` usando el ARN del bucket como referencia directa.

**Ejercicio B — templatefile con policy externa:** Extraer la policy a `policies/s3_readonly.json.tftpl` con placeholders `${bucket_arn}` y `${account_id}`. Renderizarla con `templatefile()` pasando los valores desde locals. Discutir cómo versionar las policies junto al código en Git.

**Ejercicio C — aws_iam_policy_document y composición:** Policy base con acceso de lectura y policy adicional que añade escritura solo en `prod`. Combinarlas con `source_policy_documents` para generar la policy final según el entorno.

**Ejercicio D — SSM y Secrets Manager:** Guardar un valor en SSM Parameter Store, leerlo con data source, marcarlo como `sensitive`. Ver que no aparece en el plan pero sí en el state. Discutir implicaciones y estrategias de mitigación.

---

## Módulo 16 — Aprovisionamiento y despliegue inicial

**Conceptos:** Terraform gestiona infraestructura, no configuración de software. Mecanismos disponibles dentro del propio Terraform:

`user_data`: script que ejecuta la instancia en el primer arranque. No se re-ejecuta si cambia a menos que se fuerce recreación. `templatefile()` para generar el script dinámicamente con variables.

`file` provisioner: copia ficheros desde la máquina que ejecuta Terraform a la instancia (claves SSH adicionales, configuraciones). Requiere conectividad en el momento del apply.

`remote-exec` provisioner: ejecuta comandos en la instancia vía SSH tras crearla. Útil para bootstrapping cuando `user_data` no es suficiente: esperar a que un servicio esté listo, ejecutar un script con outputs de Terraform como parámetros.

`local-exec` provisioner: ejecuta un comando en la máquina local tras crear o destruir un recurso. Casos de uso: invocar Ansible, registrar el servidor en un inventario, notificar a un sistema externo.

`terraform_data` (>= 1.4): recurso sin representación real en AWS que sirve de contenedor para provisioners no asociados a ningún recurso concreto. Sustituye al `null_resource` sin necesitar provider adicional. El campo `triggers_replace` fuerza su recreación (y por tanto re-ejecuta sus provisioners) cuando los valores cambian:

```hcl
resource "terraform_data" "run_migration" {
  triggers_replace = [var.db_version]

  provisioner "local-exec" {
    command = "python scripts/migrate.py ${aws_db_instance.main.endpoint}"
  }
}
```

Casos de uso: ejecutar una migración de base de datos tras crear o actualizar RDS, invalidar caché de CloudFront tras un despliegue, ejecutar Ansible pasándole la IP recién creada.

Bloque `connection`: cómo Terraform se conecta a la instancia (usuario, clave privada, bastion host).

Cuándo no usar provisioners: si el equipo ya usa Ansible u otro sistema de configuración, Terraform debe limitarse a crear la infraestructura y pasar los datos necesarios vía outputs. Los provisioners son un escape hatch, no el patrón principal.

**Ejercicio A — user_data con templatefile:** EC2 que recibe como variable el nombre del entorno y genera una página nginx personalizada con `templatefile()`. Verificar en el navegador.

**Ejercicio B — file + remote-exec:** Copiar una clave SSH pública adicional a `~/.ssh/authorized_keys` con `file` provisioner. Usar `remote-exec` para instalar y arrancar un servicio esperando a que el puerto esté disponible.

**Ejercicio C — local-exec:** Tras crear la EC2, ejecutar un `local-exec` que guarde la IP pública en un fichero local de inventario. Discutir por qué esto es frágil y cuándo tiene sentido.

**Ejercicio D — terraform_data como trigger:** Simular una migración de base de datos: `terraform_data` con `triggers_replace` apuntando a la versión de la app. Cambiar la versión y observar que el provisioner se re-ejecuta en el siguiente apply sin recrear ningún recurso de infraestructura real.

---

## Módulo 17 — Proyecto integrador

**Ejercicio final:** VPC con subnets calculadas con `cidrsubnet` + EC2 con nginx configurado vía `templatefile` + bucket S3 con policy gestionada con `jsonencode` + zona Route53 delegada desde `prod` a `pre` + outputs útiles. Todo organizado en módulos reutilizables, desplegado en `dev` y `prod` con state remoto, variables diferenciadas por entorno, `default_tags` en el provider, repositorio Git limpio con `.gitignore` correcto y `.terraform.lock.hcl` commiteado. Aplicar, verificar que funciona, destruir limpiamente.

---

## Nota de costes

Destruir siempre al acabar cada ejercicio. Los únicos recursos con coste real en este curso son NAT Gateway (no se usa), Elastic IPs no asociadas y EC2 fuera del free tier. Todo lo demás (S3, DynamoDB on-demand, VPC, subnets, SGs, Route53 hosted zones a ~$0.50/mes) es gratuito o de coste despreciable. Con disciplina en `terraform destroy` el coste total del curso debería ser cercano a cero.
