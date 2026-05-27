# Terraform en AWS — Curso práctico

Repositorio base del curso. Cada módulo tiene dos tags:

- `module-N` — estado inicial del ejercicio, punto de partida para el alumno
- `practice-N` — ejercicio resuelto, referencia si te atascas

## Cómo trabajar con este repo

```bash
# Clonar el repo
git clone <url-del-repo>
cd terraform-curso

# Situarte en el módulo que toca
git checkout module-0

# Crear tu rama de trabajo desde ese punto
git checkout -b mi-modulo-0
```

Cuando quieras ver la solución:

```bash
git diff module-0 practice-0
```

---

## Módulo 0 — Entorno, provider y primer commit

### Conceptos

**`terraform fmt`** — formateador oficial. Estandariza indentación y estilo. En equipos se enforza con un pre-commit hook para que nunca llegue código sin formatear al repositorio. Ejecutar siempre antes de commitear.

**`terraform validate`** — valida la sintaxis y coherencia interna del código sin conectarse a AWS. No detecta todos los errores posibles pero sí los más básicos: referencias rotas, tipos incorrectos, bloques mal formados.

**`.terraform.lock.hcl`** — fichero de lock generado por `terraform init`. Fija las versiones exactas de los providers con sus hashes criptográficos. Garantiza que todos los miembros del equipo y el CI usan exactamente el mismo binario de provider. Va a Git. Se actualiza de forma controlada con `terraform providers lock`.

**`terraform plan -out=planfile` + `terraform apply planfile`** — flujo correcto en entornos reales: el plan se genera, se revisa (o se adjunta a un PR para aprobación), y el apply ejecuta exactamente ese plan ya aprobado. Evita que el estado de AWS cambie entre el momento del plan y el del apply.

**Provider y versiones** — el bloque `required_providers` en `versions.tf` fija qué provider se usa y en qué rango de versiones. `~> 5.0` significa "cualquier versión >= 5.0 y < 6.0": acepta actualizaciones de minor y patch pero no saltos de major que podrían romper el código.

**Perfil AWS** — las credenciales de AWS se gestionan con perfiles nombrados en `~/.aws/credentials`. Usar un perfil propio por proyecto (`terraform-curso`) en lugar del perfil `default` evita operar accidentalmente sobre la cuenta o región equivocada.

### Ejercicio

Prerrequisitos: tener instalados AWS CLI y Terraform (ver instrucciones en [INSTALL.md](./INSTALL.md)).

**1. Configura el perfil AWS del curso**

```bash
aws configure --profile terraform-curso
```

Valores:
- **AWS Access Key ID:** el de tu usuario IAM `terraform-curso`
- **AWS Secret Access Key:** el de tu usuario IAM `terraform-curso`
- **Default region:** `eu-west-1`
- **Default output format:** `json`

Verificar:

```bash
aws sts get-caller-identity --profile terraform-curso
```

Exportar el perfil para no pasarlo en cada comando:

```bash
export AWS_PROFILE=terraform-curso
```

**2. Crea tu rama de trabajo**

```bash
git checkout module-0
git checkout -b mi-modulo-0
```

**3. Crea los ficheros del proyecto**

El repo ya tiene la estructura base. Edita `versions.tf` para configurar el provider:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = var.region
  profile = "terraform-curso"
}
```

Edita `variables.tf` para declarar la variable de región:

```hcl
variable "region" {
  description = "Región de AWS donde desplegar los recursos"
  type        = string
  default     = "eu-west-1"
}
```

Dejar `main.tf` y `outputs.tf` vacíos por ahora.

**4. Inicializa el proyecto**

```bash
terraform init
```

Observar que se generan:
- `.terraform/` — binarios del provider descargado. No va a Git, está en `.gitignore`.
- `.terraform.lock.hcl` — lock de versiones. Va a Git.

Ver el contenido del lock file y localizar la versión fijada y los hashes:

```bash
cat .terraform.lock.hcl
```

**5. Valida y formatea**

```bash
terraform fmt
terraform validate
# Success! The configuration is valid.
```

**6. Verifica el ciclo completo**

```bash
terraform plan
# No changes. Your infrastructure matches the configuration.
```

Con `main.tf` vacío no hay nada que crear. Si el plan no devuelve errores, el entorno está correctamente configurado.

**7. Commitea**

```bash
git add .
git status
```

Antes de commitear, verificar que en el staging area están:
- `.terraform.lock.hcl`
- `versions.tf`
- `variables.tf`

Y que **no están**:
- `.terraform/`
- Ningún `*.tfstate`

```bash
git commit -m "feat(module-0): configurar provider aws y estructura base"
```

---

### Estructura del proyecto al terminar este módulo

```
terraform-curso/
├── .gitignore
├── .terraform.lock.hcl   ← generado por terraform init, va a Git
├── README.md
├── INSTALL.md
├── main.tf               ← vacío por ahora
├── modules/
├── outputs.tf            ← vacío por ahora
├── variables.tf
└── versions.tf
```