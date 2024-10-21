# Desafio de Infraestrutura como Código (IaC) com Terraform

## Objetivo
Demonstrar conhecimentos em Infraestrutura como Código (IaC) utilizando Terraform, bem como habilidades em segurança e automação de configuração de servidores.

## Análise Técnica

O arquivo `main.tf` configura a criação de uma infraestrutura básica na AWS. Abaixo está uma descrição detalhada dos recursos definidos:

### 1. Provider
Define o provedor AWS e a região onde os recursos serão criados (`us-east-1`).

### 2. Variáveis
Define duas variáveis: `projeto` e `candidato`, que permitem nomear os recursos de forma dinâmica.

### 3. Chave SSH
Gera uma chave privada TLS e cria um par de chaves SSH para acesso à instância EC2.

### 4. VPC
Cria uma VPC (Virtual Private Cloud) com um bloco CIDR de `10.0.0.0/16` e habilita suporte a DNS.

### 5. Subnet
Cria uma sub-rede dentro da VPC, com um bloco CIDR de `10.0.1.0/24`.

### 6. Internet Gateway
Cria um gateway de internet para permitir que a VPC tenha acesso à internet.

### 7. Tabela de Roteamento
Cria uma tabela de roteamento que direciona o tráfego para o gateway de internet e associa essa tabela à sub-rede.

### 8. Grupo de Segurança
Define um grupo de segurança que permite conexões SSH de qualquer lugar e permite todo o tráfego de saída.

### 9. AMI
Obtém a imagem mais recente do Debian 12 na AWS, filtrando por nome e tipo de virtualização.

### 10. Instância EC2
Cria uma instância EC2 utilizando a AMI do Debian 12, com um tipo de instância `t2.micro`. A instância executa um script de `user_data` para atualizações de sistema.

### 11. Saídas
Define saídas que disponibilizam a chave privada gerada e o IP público da instância EC2.

### Observações
- O grupo de segurança permite acesso SSH de qualquer endereço IP, o que pode ser um risco.
- O script de inicialização poderia ser melhorado com a instalação automática do Nginx.

---

## Modificação e Melhoria do Código Terraform

### Melhorias Implementadas

#### 1. Melhoria de Segurança
- **Restrição do Acesso SSH**: O grupo de segurança foi modificado para permitir acesso SSH apenas de endereços IP específicos.

```hcl
ingress {
  description      = "Allow SSH from specific IP"
  from_port        = 22
  to_port          = 22
  protocol         = "tcp"
  cidr_blocks      = ["IPS específicos aq..."]
}
```

#### 2. Automação da Instalação do Nginx
- **Adição ao Script de User Data**: O script `user_data` foi atualizado para instalar e iniciar o Nginx automaticamente.

```hcl
user_data = <<-EOF
            #!/bin/bash
            apt-get update -y
            apt-get upgrade -y
            apt-get install nginx -y
            systemctl start nginx
            systemctl enable nginx
            EOF
```

### Descrição Técnica das Alterações
- **Segurança**: A restrição do acesso SSH aumenta a segurança da instância, limitando potenciais ataques.
- **Automação do Nginx**: A instalação automática do Nginx permite que a instância esteja pronta para uso imediato, sem necessidade de configuração manual após o provisionamento.

### Outras Melhorias
- **Tagging Consistente**: As tags foram padronizadas para facilitar o gerenciamento e identificação dos recursos, melhorando a organização.

### Descrição Técnica
O código original define uma infraestrutura na AWS, incluindo VPC, Subnet, Grupo de Segurança, Key Pair e uma instância EC2. As melhorias implementadas visam aumentar a segurança e a eficiência do provisionamento, permitindo uma configuração mais robusta e automatizada.

### main.tf Modificado

```hcl
provider "aws" {
  region = "us-east-1"
}

variable "projeto" {
  description = "Nome do projeto"
  type        = string
  default     = "VExpenses"
}

variable "candidato" {
  description = "Nome do candidato"
  type        = string
  default     = "SeuNome"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "aws_key_pair" "ec2_key_pair" {
  key_name   = "${var.projeto}-${var.candidato}-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "aws_vpc" "main_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.projeto}-${var.candidato}-vpc"
  }
}

resource "aws_subnet" "main_subnet" {
  vpc_id            = aws_vpc.main_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "${var.projeto}-${var.candidato}-subnet"
  }
}

resource "aws_internet_gateway" "main_igw" {
  vpc_id = aws_vpc.main_vpc.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-igw"
  }
}

resource "aws_route_table" "main_route_table" {
  vpc_id = aws_vpc.main_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main_igw.id
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table"
  }
}

resource "aws_route_table_association" "main_association" {
  subnet_id      = aws_subnet.main_subnet.id
  route_table_id = aws_route_table.main_route_table.id

  tags = {
    Name = "${var.projeto}-${var.candidato}-route_table_association"
  }
}

resource "aws_security_group" "main_sg" {
  name        = "${var.projeto}-${var.candidato}-sg"
  description = "Permitir SSH de IP específico e todo o tráfego de saída"
  vpc_id      = aws_vpc.main_vpc.id

  ingress {
    description      = "Allow SSH from specific IP"
    from_port        = 22
    to_port          = 22
    protocol         = "tcp"
    cidr_blocks      = ["IPS específicos aq..."]
  }

  egress {
    description      = "Allow all outbound traffic"
    from_port        = 0
    to_port          = 0
    protocol         = "-1"
    cidr_blocks      = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.projeto}-${var.candidato}-sg"
  }
}

data "aws_ami" "debian12" {
  most_recent = true

  filter {
    name   = "name"
    values = ["debian-12-amd64-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  owners = ["679593333241"]
}

resource "aws_instance" "debian_ec2" {
  ami             = data.aws_ami.debian12.id
  instance_type   = "t2.micro"
  subnet_id       = aws_subnet.main_subnet.id
  key_name        = aws_key_pair.ec2_key_pair.key_name
  security_groups = [aws_security_group.main_sg.name]

  associate_public_ip_address = true

  root_block_device {
    volume_size           = 20
    volume_type           = "gp2"
    delete_on_termination = true
  }

  user_data = <<-EOF
              #!/bin/bash
              apt-get update -y
              apt-get upgrade -y
              apt-get install nginx -y
              systemctl start nginx
              systemctl enable nginx
              EOF

  tags = {
    Name = "${var.projeto}-${var.candidato}-ec2"
  }
}

output "private_key" {
  description = "Chave privada para acessar a instância EC2"
  value       = tls_private_key.ec2_key.private_key_pem
  sensitive   = true
}

output "ec2_public_ip" {
  description = "Endereço IP público da instância EC2"
  value       = aws_instance.debian_ec2.public_ip
}
```

### Instruções de Uso

#### Pré-requisitos:
- Instalar o Terraform.
- Configurar as credenciais da AWS (AWS Access Key e Secret Key).

#### Comandos:
```bash
terraform init
terraform plan
terraform apply
```

#### Acessando a Instância:
Use o IP público da saída `ec2_public_ip` junto com a chave privada gerada para acessar a instância via SSH.

### Conclusão
As alterações realizadas melhoram a segurança e a funcionalidade do ambiente provisionado, seguindo melhores práticas de infraestrutura como código e automação.

## Autor
**Bernardo Di Domenico Ferrari**




