# Introducción a Terraform con AWS

## 1. Servicios de AWS utilizados

### Amazon S3 (Simple Storage Service)
- Servicio de almacenamiento de objetos altamente escalable
- Utilizado para almacenar y recuperar cualquier cantidad de datos
- En Terraform, se usa comúnmente como backend para almacenar el estado de la infraestructura

### Amazon DynamoDB
- Servicio de base de datos NoSQL totalmente administrado
- Ofrece rendimiento rápido y predecible con escalabilidad fluida
- En Terraform, se utiliza para implementar bloqueos de estado, evitando conflictos en operaciones concurrentes

### Amazon EC2 (Elastic Compute Cloud)
- Servicio de computación en la nube
- Proporciona capacidad de cómputo redimensionable en la nube
- En nuestro código, se utiliza indirectamente a través de configuraciones de lanzamiento y grupos de autoescalado

### Amazon VPC (Virtual Private Cloud)
- Permite lanzar recursos de AWS en una red virtual aislada lógicamente
- Proporciona control sobre el entorno de red virtual
- En nuestro código, se utiliza para definir la red en la que se desplegarán nuestros recursos

### Elastic Load Balancing (ELB)
- Distribuye automáticamente el tráfico entrante de aplicaciones entre múltiples destinos
- En nuestro código, utilizamos un Application Load Balancer (ALB)

## 2. Recursos de Terraform y su relación con AWS

### aws_launch_configuration
- Define la configuración para las instancias EC2 que se lanzarán en el grupo de autoescalado
- Especifica:
  - AMI (Amazon Machine Image)
  - Tipo de instancia
  - Grupos de seguridad
  - Script de datos de usuario para configuración inicial

### aws_autoscaling_group
- Mantiene un número específico de instancias EC2 en ejecución
- Puede escalar automáticamente basado en métricas definidas
- En nuestro código:
  - Utiliza la configuración de lanzamiento definida anteriormente
  - Define tamaños mínimo y máximo del grupo
  - Asocia las instancias con un grupo objetivo del balanceador de carga

### aws_security_group
- Define reglas de firewall virtual para controlar el tráfico entrante y saliente
- En nuestro código, definimos dos grupos de seguridad:
  1. Para las instancias EC2
  2. Para el balanceador de carga

### aws_lb (Application Load Balancer)
- Actúa como punto de entrada único para el tráfico de la aplicación
- Distribuye el tráfico entre las instancias del grupo de autoescalado

### aws_lb_listener y aws_lb_listener_rule
- Configura cómo el balanceador de carga debe manejar las solicitudes entrantes
- Define reglas para enrutar el tráfico a grupos objetivo específicos

### aws_lb_target_group
- Define un grupo de recursos (en este caso, instancias EC2) a los que el balanceador de carga puede dirigir el tráfico
- Configura comprobaciones de salud para asegurar que solo las instancias saludables reciban tráfico

### data sources (aws_vpc, aws_subnets, terraform_remote_state)
- Permite a Terraform obtener información sobre recursos existentes en AWS o estados remotos
- Útil para referenciar recursos que no son manejados directamente por el código actual de Terraform

En las siguientes secciones, profundizaremos en cada uno de estos recursos y cómo se configuran en Terraform para crear una infraestructura robusta y escalable en AWS.


## 3. Configuración del Backend de Terraform

Antes de comenzar con la infraestructura principal, es crucial configurar un backend remoto para Terraform. Esto permite almacenar el estado de forma segura y facilita la colaboración en equipo.

### Configuración del proveedor y versiones

```hcl
terraform {
  required_version = ">= 1.0.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-2"
}
```

- Especificamos la versión de Terraform y del proveedor AWS
- Definimos la región de AWS donde se crearán los recursos

### Creación del bucket S3

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = var.bucket_name
  force_destroy = true
}
```

- Creamos un bucket S3 para almacenar el estado de Terraform
- `force_destroy = true` permite eliminar el bucket incluso si contiene objetos (no recomendado para producción)

### Configuración del versionado del bucket

```hcl
resource "aws_s3_bucket_versioning" "enabled" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

- Habilitamos el versionado para mantener un historial completo de los archivos de estado

### Configuración de la encriptación del bucket

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "default" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

- Habilitamos la encriptación del lado del servidor para proteger los datos almacenados

### Bloqueo de acceso público al bucket

```hcl
resource "aws_s3_bucket_public_access_block" "public_access" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

- Bloqueamos explícitamente todo el acceso público al bucket por razones de seguridad

### Creación de la tabla DynamoDB para bloqueos

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = var.table_name
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

- Creamos una tabla DynamoDB para gestionar los bloqueos de estado
- Utilizamos el modo de facturación "PAY_PER_REQUEST" para optimizar costos
- Definimos "LockID" como la clave primaria de la tabla

Esta configuración del backend es crucial para un flujo de trabajo seguro y colaborativo con Terraform. Proporciona:

1. Almacenamiento seguro del estado en S3
2. Versionado para mantener un historial de cambios
3. Encriptación para proteger datos sensibles
4. Bloqueo de acceso público para mayor seguridad
5. Mecanismo de bloqueo con DynamoDB para prevenir conflictos en operaciones concurrentes

En las siguientes secciones, utilizaremos este backend para gestionar nuestra infraestructura principal.


## 4. Creación de la Base de Datos

Después de configurar el backend, el siguiente paso es crear la base de datos. Utilizaremos Amazon RDS (Relational Database Service) para esto.

### Configuración del proveedor y backend

```hcl
terraform {
  required_version = ">= 1.0.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }

  backend "s3" {
    bucket         = "bucket-test-luisgonzalezbernal"
    key            = "stage/terraform.tfstate"
    region         = "us-east-2"
    dynamodb_table = "table-test"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-2"
}
```

- Especificamos la versión de Terraform y del proveedor AWS
- Configuramos el backend S3 para almacenar el estado
- Definimos la región de AWS donde se crearán los recursos

### Creación de la instancia de base de datos

```hcl
resource "aws_db_instance" "example" {
  identifier_prefix   = "terraform-up-and-running"
  engine              = "mysql"
  allocated_storage   = 10
  instance_class      = "db.t3.micro"
  skip_final_snapshot = true

  db_name             = var.db_name

  username = var.db_username
  password = var.db_password
}
```

Explicación de los parámetros:

- `identifier_prefix`: Prefijo para el identificador único de la instancia de base de datos
- `engine`: Motor de base de datos a utilizar (en este caso, MySQL)
- `allocated_storage`: Espacio de almacenamiento asignado en GB
- `instance_class`: Tipo de instancia que determina la capacidad de cómputo y memoria
- `skip_final_snapshot`: Evita la creación de una instantánea final al destruir la base de datos (no recomendado para producción)
- `db_name`: Nombre de la base de datos inicial
- `username` y `password`: Credenciales para el usuario administrador de la base de datos

Notas importantes:

1. Seguridad: Las credenciales se pasan como variables para evitar exponer información sensible en el código.
2. Tamaño y rendimiento: La instancia `db.t3.micro` y 10GB de almacenamiento son adecuados para pruebas, pero pueden necesitar ajustes para casos de uso en producción.
3. Configuración del motor: Dependiendo de los requisitos específicos, pueden ser necesarias configuraciones adicionales del motor de base de datos.
4. Networking: Esta configuración asume que se está utilizando la VPC por defecto. En un entorno de producción, se debería especificar una VPC y subredes específicas.

En las siguientes secciones, veremos cómo conectar esta base de datos con el resto de nuestra infraestructura y cómo gestionar sus datos de forma segura.

## 5. Configuración de la Infraestructura Principal

Finalmente, configuraremos la infraestructura principal de nuestra aplicación, que incluye un grupo de autoescalado, un balanceador de carga y grupos de seguridad.

### Configuración del proveedor

```hcl
terraform {
  required_version = ">= 1.0.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-2"
}
```

### Configuración de Lanzamiento

```hcl
resource "aws_launch_configuration" "example" {
  image_id        = "ami-0fb653ca2d3203ac1"
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.instance.id]

  user_data = templatefile("user-data.sh", {
    server_port = var.server_port
    db_address  = data.terraform_remote_state.db.outputs.address
    db_port     = data.terraform_remote_state.db.outputs.port
  })

  lifecycle {
    create_before_destroy = true
  }
}
```

- Define la configuración para las instancias EC2 que se lanzarán
- Utiliza un template para el script de datos de usuario, pasando información de la base de datos

### Grupo de Autoescalado

```hcl
resource "aws_autoscaling_group" "example" {
  launch_configuration = aws_launch_configuration.example.name
  vpc_zone_identifier  = data.aws_subnets.default.ids

  target_group_arns = [aws_lb_target_group.asg.arn]
  health_check_type = "ELB"

  min_size = 2
  max_size = 10

  tag {
    key                 = "Name"
    value               = "terraform-asg-example"
    propagate_at_launch = true
  }
}
```

- Configura el grupo de autoescalado para mantener entre 2 y 10 instancias
- Utiliza la configuración de lanzamiento definida anteriormente

### Grupos de Seguridad

```hcl
resource "aws_security_group" "instance" {
  name = var.instance_security_group_name

  ingress {
    from_port   = var.server_port
    to_port     = var.server_port
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "alb" {
  name = var.alb_security_group_name

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

- Define reglas de firewall para las instancias y el balanceador de carga

### Balanceador de Carga de Aplicaciones (ALB)

```hcl
resource "aws_lb" "example" {
  name               = var.alb_name
  load_balancer_type = "application"
  subnets            = data.aws_subnets.default.ids
  security_groups    = [aws_security_group.alb.id]
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.example.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "fixed-response"

    fixed_response {
      content_type = "text/plain"
      message_body = "404: page not found"
      status_code  = 404
    }
  }
}

resource "aws_lb_target_group" "asg" {
  name     = var.alb_name
  port     = var.server_port
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.default.id

  health_check {
    path                = "/"
    protocol            = "HTTP"
    matcher             = "200"
    interval            = 15
    timeout             = 3
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }
}

resource "aws_lb_listener_rule" "asg" {
  listener_arn = aws_lb_listener.http.arn
  priority     = 100

  condition {
    path_pattern {
      values = ["*"]
    }
  }

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.asg.arn
  }
}
```

- Configura un ALB para distribuir el tráfico entre las instancias
- Define un listener para manejar el tráfico HTTP
- Configura un grupo objetivo y reglas de enrutamiento

### Data Sources

```hcl
data "terraform_remote_state" "db" {
  backend = "s3"

  config = {
    bucket = var.db_remote_state_bucket
    key    = var.db_remote_state_key
    region = "us-east-2"
  }
}

data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}
```

- Recupera información sobre la base de datos del estado remoto
- Obtiene información sobre la VPC y subredes por defecto

Este script final crea una infraestructura completa y escalable en AWS, incluyendo:

1. Un grupo de autoescalado para manejar la carga de la aplicación
2. Un balanceador de carga para distribuir el tráfico
3. Grupos de seguridad para controlar el acceso
4. Conexión con la base de datos creada anteriormente

En conjunto, estos recursos forman una arquitectura robusta y flexible para hospedar aplicaciones web.
