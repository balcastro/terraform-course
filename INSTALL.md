# Preparar el entorno

Instrucciones para Linux (Ubuntu/Debian). Solo necesitas hacer esto una vez antes de empezar el módulo 0.

---

## 1. Instalar AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o /tmp/awscliv2.zip
unzip /tmp/awscliv2.zip -d /tmp
sudo /tmp/aws/install
```

Verificar:

```bash
aws --version
# aws-cli/2.x.x Python/3.x.x Linux/...
```

---

## 2. Instalar Terraform

```bash
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt-get update && sudo apt-get install -y terraform
```

Verificar:

```bash
terraform version
# Terraform v1.x.x
```

---

## 3. Crear el usuario IAM para el curso

El objetivo es no usar las credenciales de root ni las de tu usuario personal. Creamos un usuario dedicado con los permisos justos para el curso.

### 3.1 Acceder a la consola de AWS

Ir a **IAM → Users → Create user**.

- **User name:** `terraform-user`
- No marcar acceso a consola — solo necesitamos acceso programático.

### 3.2 Asignar permisos

En el paso de permisos, seleccionar **Attach policies directly** y añadir `AdministratorAccess`.

> En un entorno real nunca se daría AdministratorAccess a Terraform. Para el curso es lo pragmático: evita tener que ir añadiendo permisos módulo a módulo. Al terminar el curso, este usuario se elimina.

### 3.3 Crear la access key

Una vez creado el usuario, ir a **Security credentials → Access keys → Create access key**.

- Caso de uso: **Command Line Interface (CLI)**
- Guardar el **Access key ID** y el **Secret access key** — solo se muestran una vez.

---

## 4. Exportar el perfil en tu sesión

Para no tener que pasar `--profile terraform-user` en cada comando durante el curso, añadir al final de `~/.bashrc` o `~/.zshrc`:

```bash
echo 'export AWS_PROFILE=terraform-user' >> ~/.bashrc
source ~/.bashrc
```

Verificar que todo está en orden:

```bash
aws sts get-caller-identity
# {
#     "UserId": "...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/terraform-user"
# }
```

---

A partir de aquí vuelve al README y empieza el módulo 0.