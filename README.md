# Desafio 2 — Arquitetura WSL 2 conectado à AWS

![WSL 2](https://img.shields.io/badge/WSL%202-Ubuntu%2024.04%20LTS-E95420?logo=ubuntu&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Desktop%20%2B%20Compose-2496ED?logo=docker&logoColor=white)
![AWS CLI](https://img.shields.io/badge/AWS%20CLI-v2-232F3E?logo=amazonwebservices&logoColor=white)
![EC2](https://img.shields.io/badge/Amazon-EC2-FF9900?logo=amazonec2&logoColor=white)
![ECR](https://img.shields.io/badge/Amazon-ECR-FF9900?logo=amazonaws&logoColor=white)
![SSM](https://img.shields.io/badge/Systems%20Manager-Session%20Manager-527FFF?logo=amazonaws&logoColor=white)
![IAM](https://img.shields.io/badge/IAM-Credenciais%20Tempor%C3%A1rias-DD344C?logo=amazonaws&logoColor=white)

> **Estação de desenvolvimento Linux sobre WSL 2, autenticada na AWS por credenciais temporárias, operando uma instância EC2 via Session Manager (sem porta 22 aberta para a internet) e publicando imagens Docker no Amazon ECR.**

Este repositório documenta, passo a passo, a resolução do **Desafio 2** da Mentoria *Desafios Fundamentais* da **Formação AWS**, conduzida por **Henrylle Maia**. Mais do que uma lista de comandos, ele registra as **decisões técnicas**, os **erros reais encontrados** e como cada um foi resolvido.

---

## Índice

- [1. Objetivo](#1-objetivo)
- [2. Resultado](#2-resultado)
- [3. Arquitetura](#3-arquitetura)
  - [3.1. Visão geral: local × nuvem](#31-visão-geral-local--nuvem)
  - [3.2. Arquitetura local (WSL 2 / Docker)](#32-arquitetura-local-wsl-2--docker)
  - [3.3. Arquitetura na AWS](#33-arquitetura-na-aws)
- [4. Ambiente da máquina](#4-ambiente-da-máquina)
- [5. Os dois pilares: Security Group × IAM](#5-os-dois-pilares-security-group--iam)
- [6. Configuração local passo a passo](#6-configuração-local-passo-a-passo)
  - [6.1. Preparar a estação de trabalho](#61-preparar-a-estação-de-trabalho)
  - [6.2. Conectar o WSL à conta AWS (IAM User)](#62-conectar-o-wsl-à-conta-aws-iam-user)
  - [6.3. Evoluindo para credenciais temporárias](#63-evoluindo-para-credenciais-temporárias)
  - [6.4. Validando a conexão](#64-validando-a-conexão)
- [7. Etapa extra: subindo para a AWS](#7-etapa-extra-subindo-para-a-aws)
  - [7.1. Security Group bia-dev](#71-security-group-bia-dev)
  - [7.2. Role role-acesso-ssm e validação dos recursos](#72-role-role-acesso-ssm-e-validação-dos-recursos)
  - [7.3. Lançando a EC2 bia-dev via script](#73-lançando-a-ec2-bia-dev-via-script)
  - [7.4. Acessando a máquina: SSM × SSH](#74-acessando-a-máquina-ssm--ssh)
  - [7.5. Rodando a aplicação BIA](#75-rodando-a-aplicação-bia)
  - [7.6. Build e push da imagem para o ECR](#76-build-e-push-da-imagem-para-o-ecr)
- [8. Problemas encontrados e como resolvi](#8-problemas-encontrados-e-como-resolvi)
- [9. Checklist de entrega](#9-checklist-de-entrega)
- [10. O que aprendi](#10-o-que-aprendi)
- [11. Boas práticas de segurança neste repositório](#11-boas-práticas-de-segurança-neste-repositório)
- [12. Créditos e referências](#12-créditos-e-referências)

---

## 1. Objetivo

Montar uma estação de trabalho Linux completa e conectá-la a uma conta AWS, executando o ciclo de vida de uma aplicação conteinerizada — do ambiente local até a imagem publicada no registry da nuvem.

O desafio foi entregue em duas partes:

**Parte 1 — Fundação**

- Finalizar a preparação da máquina de trabalho (AWS CLI, SAM, Node.js, Session Manager Plugin)
- Conectar a máquina local à conta AWS através de um **usuário IAM**
- Lançar a máquina de trabalho **`bia-dev`** (EC2)
- Estabelecer conexão da máquina local para a `bia-dev` **tanto por SSH quanto por SSM**

**Parte 2 — Aplicação (Dia 1 e Dia 2 do projeto BIA)**

| Dia | Tarefa |
| --- | --- |
| Dia 1 | Lançar a máquina `bia-dev` **via script**, não pelo console |
| Dia 1 | Configurar permissões IAM **no usuário**, em vez de depender apenas da role |
| Dia 1 | Testar a comunicação com o **Amazon ECR** |
| Dia 2 | Fazer o **build** da imagem Docker a partir da máquina local |
| Dia 2 | Fazer o **push** da imagem para o ECR a partir da máquina local |

> **Adaptação assumida:** a instrução original do curso sugere uma VM tradicional (VirtualBox). Optei por **WSL 2**, e o motivo está documentado em [Ambiente da máquina](#4-ambiente-da-máquina). Todas as referências a "sua VM" neste README correspondem à minha estação **WSL 2 + Ubuntu 24.04 LTS**.

---

## 2. Resultado

| # | Entrega | Status | Onde ver |
| :-: | --- | :-: | --- |
| 1 | Estação WSL 2 com AWS CLI v2, SAM, Node.js e Session Manager Plugin | ✅ | [§6.1](#61-preparar-a-estação-de-trabalho) |
| 2 | Usuário IAM criado com policies de SSM e ECR | ✅ | [§6.2](#62-conectar-o-wsl-à-conta-aws-iam-user) |
| 3 | Migração para **credenciais temporárias** (`aws login`) | ✅ | [§6.3](#63-evoluindo-para-credenciais-temporárias) |
| 4 | Security Group `bia-dev` liberando a porta 3001 | ✅ | [§7.1](#71-security-group-bia-dev) |
| 5 | EC2 `bia-dev` lançada **por script**, com a role `role-acesso-ssm` | ✅ | [§7.3](#73-lançando-a-ec2-bia-dev-via-script) |
| 6 | Acesso à EC2 por **SSM** (zero portas abertas) e por **SSH** | ✅ | [§7.4](#74-acessando-a-máquina-ssm--ssh) |
| 7 | Aplicação BIA rodando em contêiner (`3001:8080`) | ✅ | [§7.5](#75-rodando-a-aplicação-bia) |
| 8 | Imagem Docker publicada no Amazon ECR a partir do WSL | ✅ | [§7.6](#76-build-e-push-da-imagem-para-o-ecr) |

**A aplicação no ar, servida pelo contêiner na porta 3001:**

![Aplicação BIA rodando](imagens/Lancar_maquina_de_trabalho_bia-dev.png)

> Os prints deste repositório estão **censurados** nas regiões que continham Account ID, ARNs e IDs de recursos. Ver [§11](#11-boas-práticas-de-segurança-neste-repositório).

---

## 3. Arquitetura

### 3.1. Visão geral: local × nuvem

O laboratório tem dois lados. O **local** é onde eu escrevo, construo e me autentico. O **remoto** é onde a carga roda. A fronteira entre os dois é atravessada por credenciais temporárias — nunca por chaves fixas.

```text
        LOCAL (minha máquina)                          NUVEM (AWS)
 ┌───────────────────────────────────┐        ┌───────────────────────────────────┐
 │  Windows 11                       │        │  Conta AWS — região us-east-1     │
 │   └── WSL 2 · Ubuntu 24.04 LTS    │        │                                   │
 │        ├── AWS CLI v2             │  auth  │   IAM User + Policies             │
 │        ├── Session Manager Plugin │ ─────► │   (credenciais temporárias, 12h)  │
 │        ├── Node.js                │        │                                   │
 │        ├── SAM CLI                │  ssm   │   EC2 bia-dev                     │
 │        └── Docker (via Desktop)   │ ─────► │    └── Docker · BIA (3001:8080)   │
 │                                   │        │                                   │
 │        docker build / push        │  ecr   │   Amazon ECR                      │
 └───────────────────────────────────┘ ─────► │    └── repositório "bia"          │
                                              └───────────────────────────────────┘
```

### 3.2. Arquitetura local (WSL 2 / Docker)

A diferença essencial entre uma VM tradicional e o WSL 2 é **quantos sistemas operacionais disputam a memória** e **quantas engines de contêiner existem**:

```text
      VirtualBox (VM tradicional)                    WSL 2 (adotado)
 ┌───────────────────────────────────┐      ┌───────────────────────────────────┐
 │ Windows                           │      │ Windows                           │
 │  └── VM Ubuntu (SO completo)      │      │  └── Ubuntu (kernel compartilhado)│
 │       └── Docker Engine próprio   │      │       └── Docker CLI              │
 │                                   │      │            └── Docker Desktop     │
 │  RAM   : dividida entre 2 SOs     │      │                 (engine único)    │
 │  Boot  : minutos                  │      │  RAM   : alocação dinâmica        │
 │  Windows: integração manual       │      │  Boot  : segundos                 │
 └───────────────────────────────────┘      │  Windows: /mnt/c nativo           │
                                            └───────────────────────────────────┘
```

O WSL 2 enxerga o disco do Windows nativamente — recurso usado, por exemplo, para instalar o SAM CLI a partir do `.zip` baixado pelo navegador do Windows:

```text
C:\Users\<seu-usuario>\Downloads\aws-sam-cli-linux-x86_64.zip
                    ↓  (o mesmo arquivo, visto de dentro do Linux)
/mnt/c/Users/<seu-usuario>/Downloads/aws-sam-cli-linux-x86_64.zip
```

### 3.3. Arquitetura na AWS

![Arquitetura da aplicação BIA-DEV na AWS](imagens/Arquitetura_oficial.jpg)

**Legenda dos componentes**

| Componente | Papel na arquitetura |
| --- | --- |
| **Terminal / IAM Keys** | Minha estação WSL acessando a AWS por linha de comando, autenticada pelo IAM |
| **AWS Console** | Interface web, usada apenas para conferência visual e criação inicial do usuário |
| **VPC** | Rede virtual isolada que contém todos os recursos do laboratório |
| **Subnet** | Sub-rede (zona A) onde a instância EC2 foi alocada |
| **IPv4 público** | O "X" no diagrama indica o acesso vindo da internet, bloqueado/restrito por padrão |
| **SG (Security Group)** | Firewall virtual da EC2 — libera apenas a porta **3001** |
| **BIA-DEV (EC2)** | Servidor de desenvolvimento onde a aplicação roda |
| **Container `3001:8080`** | O tráfego que chega na porta 3001 do host é redirecionado para a 8080 do contêiner |
| **IAM Role** | Identidade *vestida* pela EC2 — permite acessar serviços AWS **sem chave fixa no código** |
| **IAM Policy** | Regras anexadas à role: acesso ao **SSM** e ao **ECR** |
| **SSM** | Abre um terminal seguro na EC2 **sem depender de SSH e sem porta 22 aberta** |
| **ECR** | Registry privado de imagens Docker — destino da imagem construída no WSL |
| **ECS · S3 · EB** | Serviços do ecossistema presentes no ambiente, usados nas etapas seguintes da formação |

---

## 4. Ambiente da máquina

### Hardware

| Item | Especificação |
| --- | --- |
| Processador | Intel Core i3 · 2 núcleos |
| Memória RAM | 8 GB |
| Sistema host | Windows 11 Pro |
| Ambiente Linux | **WSL 2 · Ubuntu 24.04 LTS** |

### Por que WSL 2 e não uma VM tradicional

Essa foi a **primeira decisão técnica do desafio**, e ela nasceu de uma restrição real de recurso.

Com 8 GB de RAM e um i3 de 2 núcleos, subir um sistema operacional completo no VirtualBox significaria manter **dois SOs disputando a mesma memória**, cada um com sua própria engine de contêiner. O WSL 2 compartilha o kernel e faz alocação dinâmica de memória, e o Docker Desktop expõe uma **engine única** para os dois lados através da *WSL Integration*.

Resultado prático: o mesmo aprendizado e a mesma superfície de comandos Linux, por uma fração do custo de memória — **adaptar a ferramenta à restrição sem abrir mão do objetivo de aprendizagem**.

### Stack instalada

| Ferramenta | Para que serve neste desafio |
| --- | --- |
| **AWS CLI v2** | Executar qualquer ação na AWS pelo terminal |
| **SAM CLI** | Base para Lambda / CloudFormation / IaC nas etapas seguintes da formação |
| **Node.js** | Requisito de alguns comandos do projeto BIA |
| **Session Manager Plugin** | **Obrigatório** para abrir o terminal interativo do SSM |
| **Docker Desktop + WSL Integration** | Build e execução dos contêineres a partir do WSL |

---

## 5. Os dois pilares: Security Group × IAM

Antes de qualquer comando, é preciso entender o conceito que sustenta o desafio inteiro. Todo acesso na AWS passa por **duas camadas independentes e sequenciais**:

```text
┌──────────────────────────────────────────────────────────────┐
│  1. COMUNICAÇÃO                                              │
│     "Eu consigo chegar até o serviço?"                       │
│     VPC + Subnet + Rotas + Internet/NAT Gateway + SG         │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────────┐
│  2. AUTORIZAÇÃO                                              │
│     "Agora que cheguei, posso executar essa ação?"           │
│     IAM User + IAM Role + IAM Policy                         │
└────────────────────────────┬─────────────────────────────────┘
                             ▼
                    ┌────────────────┐
                    │ AÇÃO EXECUTADA │
                    └────────────────┘
```

| Recurso | Pergunta que responde | Regra mnemônica |
| --- | --- | --- |
| **Security Group** | Eu consigo **me comunicar** com esse recurso? | *Quem pode falar com quem?* |
| **IAM** | Eu tenho **autorização** para executar essa ação? | *Quem pode fazer o quê?* |

A ordem importa. Se o Security Group do destino não permitir a conexão, **a comunicação falha antes** de o IAM sequer ser consultado — não adianta ter a permissão certa. E o inverso também vale: chegar até o serviço não significa poder agir sobre ele.

**Identidade para pessoa × identidade para máquina**

```text
   MINHA MÁQUINA (pessoa)              RECURSO NA AWS (máquina)

   WSL / Terminal                      EC2
        │                               │  não armazena Access Key
        ▼                               ▼
   AWS CLI                           IAM ROLE
        │  usa credenciais              │
        ▼                               ▼
   IAM USER                          IAM POLICIES
        │                               ├──► permissão para SSM
        ▼                               ├──► permissão para ECR
   AWS API                              └──► permissão para outros serviços
```

Esse é o ponto central: **usuário para gente, role para máquina**. Nunca se cria um "usuário com senha" para um servidor — ele veste uma role.

---

## 6. Configuração local passo a passo

> **Convenção usada nos blocos abaixo:** o prompt `dev@wsl:~$` representa a minha estação local (WSL 2) e `[ec2-user@ip-172-31-x-x ~]$` representa a instância EC2 na AWS. Todos os identificadores de conta, ARNs, IPs públicos e IDs de recurso foram substituídos por valores de exemplo.

### 6.1. Preparar a estação de trabalho

**AWS CLI v2** — a porta de entrada para tudo:

```bash
sudo apt-get install curl unzip -y

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

aws --version
```

**SAM CLI** — usado quando o trabalho envolve Lambda, templates de CloudFormation e provisionamento de infraestrutura como código. Aqui aproveitei a integração do WSL com o sistema de arquivos do Windows:

```bash
# confirma que o .zip baixado pelo Windows está visível no Linux
ls -lh /mnt/c/Users/<seu-usuario>/Downloads/aws-sam-cli-linux-x86_64.zip

unzip /mnt/c/Users/<seu-usuario>/Downloads/aws-sam-cli-linux-x86_64.zip -d sam-installation
ls -lh sam-installation
```

**Node.js** — exigido por alguns comandos do projeto BIA:

```bash
cd ~
curl -sL https://deb.nodesource.com/setup_25.x -o nodesource_setup.sh
sudo bash nodesource_setup.sh
sudo apt install nodejs

node -v
```

**Session Manager Plugin** — este é o componente que a maioria esquece. O AWS CLI sozinho **não** abre a tela interativa do SSM; ele aciona este plugin em segundo plano:

```bash
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" \
  -o "session-manager-plugin.deb"
sudo dpkg -i session-manager-plugin.deb

aws ssm --version
```

### 6.2. Conectar o WSL à conta AWS (IAM User)

Para executar `aws ec2 describe-instances` a partir do WSL, o CLI precisa responder a duas perguntas — e elas são exatamente as duas camadas da [§5](#5-os-dois-pilares-security-group--iam):

```text
1. QUEM ESTÁ FAZENDO ESTA REQUISIÇÃO?   →  Autenticação (credenciais)
2. ESSA IDENTIDADE TEM PERMISSÃO?       →  Autorização (IAM Policy)
                                        →  Executar ou negar a ação
```

**Criando o usuário no console:**

`IAM` → `Users` → `Create user` → nome do usuário → `Next` → `Attach policies directly`, selecionando:

- `AmazonSSMFullAccess`
- `AmazonEC2ContainerRegistryFullAccess`

`Next` → `Create user`.

![Usuário IAM com as policies anexadas](imagens/Criacao_IAM_fundamentos_aws.png)

**Configurando as credenciais no terminal:**

```bash
# sem --profile, o CLI grava no "default profile"
aws configure

# com profile nomeado (recomendado, permite alternar contextos)
aws configure --profile fundamentos
```

Valores informados:

```text
AWS Access Key ID     [None]: <sua-access-key>
AWS Secret Access Key [None]: <sua-secret-key>
Default region name   [None]: us-east-1
Default output format [None]: table
```

> Note que este usuário foi criado **sem acesso ao console** — apenas credenciais programáticas. Ele existe para o terminal, não para a interface web. É o primeiro passo do princípio do menor privilégio.

### 6.3. Evoluindo para credenciais temporárias

Access Keys são credenciais de **longa duração**: quem as obtém, acessa a conta — indefinidamente. No fim de 2025 a AWS liberou um mecanismo mais seguro, e foi para ele que este laboratório migrou.

| | Access Keys (`aws configure`) | `aws login` |
| --- | --- | --- |
| Duração | Indefinida, até serem revogadas | **12 horas** |
| Rotação | Manual | **Automática, a cada 15 minutos** |
| Armazenamento local | Chave permanente em disco | Token temporário |
| Risco em caso de vazamento | Alto e persistente | Limitado à janela de validade |

**Pré-requisito:** o usuário precisa conseguir acessar o console. No `IAM` → `Create user`, marcar *Provide user access to the AWS Management Console* e anexar, além das policies anteriores:

- `AmazonSSMFullAccess`
- `AmazonEC2ContainerRegistryFullAccess`
- **`SignInLocalDevelopmentAccess`** — é esta policy que autoriza o terminal a gerar as credenciais temporárias

![Policy de acesso local anexada ao usuário](imagens/Primeiro_aws_login.png)

**Autenticando pelo terminal:**

```bash
dev@wsl:~$ aws login --profile formacao_aws
AWS Region [us-east-1]: us-east-1
```

O comando abre o navegador (ou imprime a URL para abrir manualmente) e, após a escolha da conta, devolve:

```text
Updated profile formacao_aws to use arn:aws:iam::111122223333:user/formacao_aws credentials.
Use "--profile formacao_aws" to use the new credentials.
```

Para encerrar a sessão:

```bash
aws logout --profile formacao_aws
```

> **Importante distinguir os dois papéis:** `aws login` resolve **quem eu sou** (autentica meu terminal na conta). O **SSM** resolve **como eu entro na máquina**. São problemas diferentes, resolvidos por serviços diferentes.

### 6.4. Validando a conexão

**Quem sou eu para a AWS:**

```bash
dev@wsl:~$ aws sts get-caller-identity --profile formacao_aws
{
    "UserId": "AIDAEXAMPLEUSERID123",
    "Account": "111122223333",
    "Arn": "arn:aws:iam::111122223333:user/formacao_aws"
}
```

**A permissão de ECR está de fato funcionando:**

```bash
dev@wsl:~$ aws ecr describe-repositories --region us-east-1
```

Retorno com os repositórios da conta = comunicação e autorização confirmadas de ponta a ponta.

---

## 7. Etapa extra: subindo para a AWS

### 7.1. Security Group bia-dev

Todo recurso precisa de um Security Group. Este libera a porta em que a aplicação BIA responde:

`EC2` → `Security Groups` → `Create security group`

| Campo | Valor |
| --- | --- |
| Security group name | `bia-dev` |
| Description | acesso do bia-dev |
| Inbound — Type | Custom TCP |
| Inbound — Protocol | TCP |
| Inbound — Port range | **3001** |
| Inbound — Source | Anywhere-IPv4 |

![Security Group bia-dev criado](imagens/Criacao_Security_Groups_bia-dev.png)

> Liberar `Anywhere-IPv4` é aceitável em um laboratório de porta de aplicação. Em produção, essa origem seria restrita a um Load Balancer ou a uma faixa de IPs conhecida.

### 7.2. Role role-acesso-ssm e validação dos recursos

O Security Group resolve a **comunicação**; a role resolve a **autorização** da instância para conversar com os outros serviços da AWS. Sem ela, a EC2 não fala com o SSM nem com o ECR.

O projeto BIA traz um script de validação que confere todos os pré-requisitos antes do lançamento:

```bash
dev@wsl:~/bia$ ./scripts/validar_recursos_zona_a.sh
[OK] Tudo certo com a VPC
[OK] Tudo certo com a Subnet
[OK] Security Group bia-dev foi criado
 [OK] Regra de entrada está ok
 [OK] Regra de saída está correta
[OK] Tudo certo com a role 'role-acesso-ssm'
```

Chegar nesse resultado exigiu resolver dois problemas reais — ambos documentados em [§8](#8-problemas-encontrados-e-como-resolvi).

Criando a role:

```bash
dev@wsl:~/bia$ ./scripts/criar_role_ssm.sh
```

### 7.3. Lançando a EC2 bia-dev via script

Com a validação 100% verde, a instância nasce por script — **não pelo console**. Esse é justamente o ponto do desafio: infraestrutura reproduzível.

```bash
dev@wsl:~/bia$ ./scripts/lancar_ec2_zona_a.sh
```

Localizando a instância recém-criada:

```bash
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,Name:Tags[?Key=='Name']|[0].Value,Lancamento:LaunchTime}" \
  --output table \
  --profile formacao_aws
```

```text
---------------------------------------------------------------------
|                         DescribeInstances                         |
+----------------------+-----------------------------+--------------+
|          ID          |         Lancamento          |     Name     |
+----------------------+-----------------------------+--------------+
|  i-0abcd1234efgh5678 |  2026-08-24T13:53:56+00:00  |  bia-dev-vm  |
|  i-0abcd1234efgh5679 |  2026-08-24T17:19:58+00:00  |  bia-dev-ssh |
|  i-0abcd1234efgh5680 |  2026-08-25T14:29:13+00:00  |  bia-dev     |
|  i-0abcd1234efgh5681 |  2026-08-25T15:32:24+00:00  |  bia-dev     |
+----------------------+-----------------------------+--------------+
```

![Instâncias EC2 em execução no console](imagens/Criacao_EC2_bia-dev.png)

### 7.4. Acessando a máquina: SSM × SSH

O desafio pedia **os dois caminhos**. Fiz ambos justamente para entender por que um deles é melhor.

```text
                                   +-------+
                      +----------->|  IAM  |  (autenticação / permissão)
                      |            +-------+
                      |
+-------------+       |          (OUTBOUND — o agente inicia)
| USER (SSM)  |◄──────┼─────────────────────────────+
+-------------+       |    [ SSM SESSION ]          |
                      +-----------------------------|
                                              +-----+-----------+
                                              | EC2             |
+-------------+   (INBOUND — porta 22)        |   AGENT SSM     |
| USER (SSH)  |─────────────────────────────► |                 |
|  chave .pem |      [ CONEXÃO SSH ]          |                 |
+-------------+                               +-----------------+
```

#### Caminho A — SSM (Session Manager)

**Requisitos, dos dois lados:**

| Lado | Requisito |
| --- | --- |
| Máquina local | AWS CLI configurado, com permissão para `ssm:StartSession` |
| Máquina local | **Session Manager Plugin** instalado |
| EC2 | `amazon-ssm-agent` rodando (já vem ativo no Amazon Linux 2023) |
| EC2 | Role IAM anexada, com `AmazonSSMManagedInstanceCore` |
| EC2 | Saída para a internet (subnet pública / NAT) ou VPC Endpoints |

Se qualquer uma dessas cinco pontas faltar, o terminal responde com timeout ou "instância não está online".

**Conectando:**

```bash
dev@wsl:~/bia$ aws ssm start-session --target i-0abcd1234efgh5681 --profile formacao_aws

Starting session with SessionId: formacao_aws-xxxxxxxxxxxxxxxxxxxxxxxxx

sh-5.2$ sudo su - ec2-user
[ec2-user@ip-172-31-15-143 ~]$
```

**Por que `sudo su - ec2-user` e não trabalhar em `/usr/bin`:**

- `/usr/bin` é reservado aos executáveis do sistema operacional e ao gerenciador de pacotes. Misturar código de aplicação ali gera **conflito** — uma atualização do SO pode sobrescrever arquivos.
- Escrever nessas pastas exige **root**, e rodar aplicação como root é uma falha grave de segurança.
- `ec2-user` é o usuário padrão do Amazon Linux. Em `/home/ec2-user` há controle total para clonar repositórios e instalar bibliotecas **sem risco de quebrar o servidor**.

**A prova de que a sessão é SSM** — a árvore de processos:

```bash
[ec2-user@ip-172-31-15-143 bia]$ ps -ef --forest | grep ssm
root        1427       1  /usr/bin/amazon-ssm-agent
root        1528    1427   \_ /usr/bin/ssm-agent-worker
root        3792    1528       \_ /usr/bin/ssm-session-worker formacao_aws-xxxxxxxx
ssm-user    3805    3792           \_ sh
```

A presença de **`/usr/bin/ssm-session-worker`** é a prova categórica: a conexão foi estabelecida pelo Systems Manager, **sem a porta 22 aberta para a internet**.

Hierarquia do que está acontecendo:

```text
amazon-ssm-agent  (PID 1427) ─ serviço base, escuta pedidos de conexão da AWS
   └── ssm-agent-worker (PID 1528) ─ gerencia tarefas e processos da máquina
        └── ssm-session-worker (PID 3792) ─ criado exclusivamente para a MINHA sessão
             └── sh (PID 3805) ─ shell inicial, sob o usuário 'ssm-user'
                  └── su - ec2-user ─ ambiente de trabalho seguro
```

**Para sair**, `exit` precisa ser executado mais de uma vez, porque há camadas empilhadas: `ec2-user` → `ssm-user` → terminal local.

#### Caminho B — SSH tradicional

Aqui a lógica se inverte: é preciso **abrir uma regra de entrada**.

**1. Security Group dedicado:**

`EC2` → `Security Groups` → `Create security group`

| Campo | Valor |
| --- | --- |
| Security group name | `bia-dev-ssh` |
| Description | acesso por ssh |
| Inbound — Type | SSH |
| Inbound — Source | **My IP** |

**2. Instância com IPv4 público** — sem ele não há como alcançar a máquina de fora.

**3. Permissão da chave privada.** O SSH é rigoroso: se o `.pem` puder ser lido por qualquer usuário do sistema, a conexão é **recusada**.

```bash
# localizar a chave
find / -name "formacao.pem" 2>/dev/null

# restringir a leitura ao meu usuário
chmod 400 /home/dev/laboratorios-aws/formacaoaws/bia/formacao.pem
```

**4. Conectar:**

```bash
dev@wsl:~$ ssh -i "/home/dev/laboratorios-aws/formacaoaws/bia/formacao.pem" \
  ec2-user@ec2-203-0-113-10.compute-1.amazonaws.com

   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'

[ec2-user@ip-172-31-4-102 ~]$ pwd
/home/ec2-user
```

**A prova de que a sessão é SSH:**

```bash
[ec2-user@ip-172-31-4-102 ~]$ echo $SSH_CLIENT
203.0.113.45 53650 22        # IP de origem, porta local, porta de destino 22

[ec2-user@ip-172-31-4-102 ~]$ who am i
[ec2-user@ip-172-31-4-102 ~]$ hostname -I     # lista os IPs das interfaces ativas
[ec2-user@ip-172-31-4-102 ~]$ whoami
```

A porta **22** no `$SSH_CLIENT` é o carimbo: essa sessão entrou pelo SSH, não pelo SSM.

#### O comparativo

| Característica | SSH tradicional | AWS SSM (Session Manager) |
| --- | --- | --- |
| Autenticação | Arquivo de chave `.pem` | **AWS IAM** (usuários / roles) |
| Portas de entrada | Requer a **porta 22 aberta** | **Zero portas abertas** |
| Direção do tráfego | *Inbound* (entra na EC2) | *Outbound* (a EC2 chama o SSM) |
| Necessidade de IP público | Sim (ou bastion host / VPN) | Não — pode ser 100% privada |
| Gestão de chaves | Manual, com compartilhamento de arquivo | Centralizada pelo IAM |
| Auditoria e logs | Logs locais no Linux | **CloudTrail, S3 e CloudWatch** |
| Superfície de ataque | Alta (porta exposta na internet) | **Mínima** |
| Suporte a MFA / SSO | Não nativo | Nativo, via IAM |

**A inversão que explica tudo:** no SSH, *eu* bato na porta do servidor — por isso preciso abri-la. No SSM, o **agente dentro da EC2** é quem inicia a comunicação de saída com o serviço da AWS. Não há porta para abrir, e não há porta para atacar.

### 7.5. Rodando a aplicação BIA

Já dentro da instância, como `ec2-user`:

```bash
[ec2-user@ip-172-31-15-143 ~]$ git clone https://github.com/henrylle/bia
[ec2-user@ip-172-31-15-143 ~]$ cd bia
[ec2-user@ip-172-31-15-143 bia]$ docker compose up -d
[ec2-user@ip-172-31-15-143 bia]$ docker compose ps
```

Três contêineres no ar:

| NAME | IMAGE | STATUS | PORTS |
| --- | --- | --- | --- |
| `bia` | `bia-server` | Up | `0.0.0.0:3001->8080/tcp` |
| `database` | `postgres:17.1` | Up | `0.0.0.0:5433->5432/tcp` |
| `redis` | `valkey/valkey:8.1-alpine` | Up | `0.0.0.0:6379->6379/tcp` |

A porta **3001** liberada no Security Group ([§7.1](#71-security-group-bia-dev)) é exatamente a que o contêiner `bia` publica — e é por ela que a aplicação responde:

![Aplicação BIA no ar](imagens/Lancar_maquina_de_trabalho_bia-dev.png)

### 7.6. Build e push da imagem para o ECR

Esta é a entrega do Dia 2: construir a imagem **na minha máquina local (WSL)** e publicá-la no registry privado da AWS.

**1. Criar o repositório no ECR:**

Pelo console: `ECR` → `Create repository` → nome `bia`. Ou pelo CLI:

```bash
aws ecr create-repository --repository-name bia --region us-east-1 --profile formacao_aws
```

**2. Habilitar o Docker no WSL** (uma vez, no Docker Desktop):

`Settings` → `Resources` → `WSL Integration` → ativar a distro Ubuntu → `Apply & restart`.

**3. Autenticar o Docker local no ECR:**

```bash
dev@wsl:~/bia$ aws ecr get-login-password --region us-east-1 --profile formacao_aws \
  | docker login --username AWS --password-stdin 111122223333.dkr.ecr.us-east-1.amazonaws.com

Login Succeeded
```

**4. Build, tag e push:**

```bash
docker build -t bia .
docker tag bia:latest 111122223333.dkr.ecr.us-east-1.amazonaws.com/bia:latest
docker push 111122223333.dkr.ecr.us-east-1.amazonaws.com/bia:latest
```

> O projeto BIA também traz um script pronto para isso — basta copiá-lo e apontar para o seu registry:
>
> ```bash
> cp scripts/ecs/unix/build.sh .
> nano build.sh    # trocar ECR_REGISTRY="SEU_REGISTRY" pela URI do seu ECR
> chmod +x build.sh
> ./build.sh
> ```

**5. Conferir o resultado:**

```bash
aws ecr describe-repositories --region us-east-1
```

```json
{
    "repositories": [
        {
            "repositoryArn": "arn:aws:ecr:us-east-1:111122223333:repository/bia",
            "registryId": "111122223333",
            "repositoryName": "bia",
            "repositoryUri": "111122223333.dkr.ecr.us-east-1.amazonaws.com/bia",
            "imageTagMutability": "MUTABLE",
            "imageScanningConfiguration": { "scanOnPush": false },
            "encryptionConfiguration": { "encryptionType": "AES256" }
        }
    ]
}
```

![Repositório bia no Amazon ECR](imagens/Criacao_repositorio_ECR.png)

---

## 8. Problemas encontrados e como resolvi

Esta seção é, para mim, a mais valiosa do repositório: nenhum desses erros aparece nos tutoriais.

### 8.1. O script não encontrava VPC, subnet nem Security Group

```text
dev@wsl:~/bia$ ./scripts/validar_recursos_zona_a.sh
[ERRO] Tenho um problema ao retornar a VPC default. Será se ela existe?
[ERRO] Tenho um problema ao retornar a subnet da zona a.
[ERRO] Não achei o security group bia-dev. Ele foi criado?
[ERRO] A role 'role-acesso-ssm' não existe
```

**Diagnóstico:** os recursos existiam. O script chamava `aws ec2 ...` **sem `--profile`**, então o CLI usava o profile *default* — que estava vazio, apontando para lugar nenhum.

**Solução:** promover o profile autenticado a padrão da sessão do shell, antes de rodar qualquer script:

```bash
export AWS_PROFILE=formacao_aws
export AWS_DEFAULT_REGION=us-east-1
```

**Aprendizado:** o AWS CLI não tem "contexto implícito". Quando um script não passa `--profile`, quem define o contexto é a variável de ambiente.

### 8.2. AccessDenied ao criar a role pelo script

```text
dev@wsl:~/bia$ ./scripts/criar_role_ssm.sh
An error occurred (AccessDenied) when calling the CreateRole operation:
User: arn:aws:iam::111122223333:user/formacao_aws is not authorized to perform:
iam:CreateRole on resource: arn:aws:iam::111122223333:role/role-acesso-ssm
because no identity-based policy allows the iam:CreateRole action
```

Os mesmos erros se repetiram para `CreateInstanceProfile`, `AddRoleToInstanceProfile` e `AttachRolePolicy`.

**Diagnóstico:** o usuário tinha permissão de **usar** SSM e ECR, mas nenhuma de **criar identidades IAM**. Este é exatamente o item do desafio *"configurar permissões IAM para o usuário ao invés da role"*.

**Solução:** anexar ao usuário uma *inline policy* (`IAM` → `Users` → usuário → `Add permissions` → `Create inline policy` → aba JSON):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateRole",
                "iam:GetRole",
                "iam:CreateInstanceProfile",
                "iam:GetInstanceProfile",
                "iam:AddRoleToInstanceProfile",
                "iam:AttachRolePolicy",
                "iam:PassRole"
            ],
            "Resource": "*"
        }
    ]
}
```

> ⚠️ **Ressalva consciente:** `"Resource": "*"` é aceitável em um laboratório descartável, mas **não em produção**. O correto seria restringir o `Resource` aos ARNs específicos da role e do instance profile, e condicionar o `iam:PassRole` ao serviço `ec2.amazonaws.com`. Deixo o registro aqui porque reconhecer o débito faz parte da entrega.

Com a policy no lugar, o script rodou e a validação fechou verde.

### 8.3. Docker invisível dentro do WSL

O `docker build` falhava porque o Docker Desktop não estava exposto para a distro Linux.

**Solução:** Docker Desktop → `Settings` → `Resources` → `WSL Integration` → marcar *Enable integration with my default WSL distro*, ativar a chave ao lado do Ubuntu → `Apply & restart`.

### 8.4. `gio: Operation not supported` no `aws login`

O `aws login` tenta abrir o navegador do host e, no WSL, isso pode falhar:

```text
gio: https://us-east-1.signin.aws.amazon.com/v1/authorize?... : Operation not supported
```

**Não é um erro fatal.** O próprio comando imprime a URL de autorização — basta copiá-la e abrir no navegador do Windows. O fluxo continua normalmente e o profile é atualizado ao final.

### 8.5. Sair do SSM exige mais de um `exit`

Ao digitar `exit` a primeira vez, o prompt vira `sh-5.2$` em vez de voltar ao WSL:

```text
[ec2-user@ip-172-31-15-143 ~]$ exit
logout
sh-5.2$
```

**Por quê:** existem camadas empilhadas. O primeiro `exit` encerra o `ec2-user` e devolve ao `ssm-user` (o usuário do agente). O segundo encerra o túnel do Systems Manager e devolve o controle ao terminal local.

---

## 9. Checklist de entrega

**Parte 1 — Fundação**

- [x] AWS CLI v2 instalado e funcional no WSL
- [x] SAM CLI instalado
- [x] Node.js instalado
- [x] Session Manager Plugin instalado (`aws ssm --version`)
- [x] Usuário IAM criado com policies de SSM e ECR
- [x] Credenciais configuradas no terminal (`aws configure`)
- [x] Migração para credenciais temporárias (`aws login` + `SignInLocalDevelopmentAccess`)
- [x] Máquina de trabalho `bia-dev` lançada
- [x] Conexão do WSL para a `bia-dev` **por SSM**, comprovada via `ssm-session-worker`
- [x] Conexão do WSL para a `bia-dev` **por SSH**, comprovada via `$SSH_CLIENT` na porta 22

**Parte 2 — Dia 1**

- [x] Security Group `bia-dev` criado (TCP 3001)
- [x] Role `role-acesso-ssm` criada por script
- [x] EC2 `bia-dev` lançada **via script** (`lancar_ec2_zona_a.sh`)
- [x] Permissões IAM configuradas **no usuário**, e não apenas na role
- [x] Comunicação com o ECR testada (`aws ecr describe-repositories`)
- [x] Aplicação BIA rodando em contêiner na porta 3001

**Parte 2 — Dia 2**

- [x] Docker habilitado no WSL via *WSL Integration*
- [x] Docker autenticado no ECR (`Login Succeeded`)
- [x] Repositório `bia` criado no ECR
- [x] `docker build` executado a partir do WSL
- [x] `docker push` da imagem para o ECR concluído

---

## 10. O que aprendi

1. **Comunicação e autorização são camadas distintas.** Security Group responde *"eu chego lá?"*; IAM responde *"eu posso agir?"*. Depurar na ordem errada custa horas.

2. **Usuário para pessoa, role para máquina.** Uma EC2 nunca deve guardar Access Key em disco — ela veste uma role e recebe credenciais rotacionadas pelo próprio serviço.

3. **Credencial temporária é o padrão, não o luxo.** Uma Access Key vazada dá acesso indefinido; um token de 12 horas com rotação a cada 15 minutos limita drasticamente a janela de dano.

4. **SSM elimina uma classe inteira de risco.** Ao inverter a direção da conexão — o agente da EC2 é quem chama a AWS — a porta 22 simplesmente deixa de existir como superfície de ataque, e todo acesso passa a ser auditável pelo CloudTrail.

5. **O profile é o contexto do CLI, e ele não é adivinhado.** Scripts que não passam `--profile` dependem de `AWS_PROFILE`. Esse detalhe foi a causa raiz do meu primeiro bloqueio.

6. **Permissão de *usar* ≠ permissão de *criar*.** Ter `AmazonSSMFullAccess` não dá o direito de criar a role que o SSM vai usar. Ler a mensagem de `AccessDenied` com atenção entrega a ação exata que falta.

7. **Onde a aplicação mora importa.** `/usr/bin` é do sistema operacional; `/home/ec2-user` é meu. Respeitar essa fronteira evita conflito de atualização e a tentação de rodar tudo como root.

8. **Restrição de recurso é um problema de arquitetura.** Trocar VirtualBox por WSL 2 não foi atalho — foi escolher a topologia que cabia no hardware disponível sem perder nenhum objetivo de aprendizagem.

---

## 11. Boas práticas de segurança neste repositório

Publicar um laboratório de nuvem exige o mesmo cuidado que operá-lo. O que foi feito antes deste repositório ir ao ar:

| Prática | Como foi aplicada |
| --- | --- |
| **Account ID mascarado** | Substituído por `111122223333` no texto e coberto por tarja em todos os prints |
| **ARNs e URIs sanitizados** | Todos os ARNs usam o Account ID de exemplo |
| **IP público de origem removido** | Substituído pela faixa de documentação `203.0.113.0/24` (RFC 5737) |
| **IDs de recurso genéricos** | Instâncias, VPC, Security Groups e Session IDs trocados por valores de exemplo |
| **Chaves privadas fora do Git** | `*.pem`, `*.ppk` e `*.csv` estão no `.gitignore` — uma chave `.pem` **nunca** deve ser versionada |
| **Anotações brutas fora do Git** | O arquivo de notas originais, que contém os dados reais, está no `.gitignore` |
| **Prints originais fora do Git** | Guardados localmente em `_originais_privados/`, também ignorado |

**Débitos reconhecidos, e que ficam como próximo passo:**

- **MFA não habilitado** no usuário IAM — visível como aviso no próprio print da [§6.2](#62-conectar-o-wsl-à-conta-aws-iam-user). Em qualquer conta com valor real, MFA é obrigatório.
- **`"Resource": "*"`** na inline policy da [§8.2](#8-problemas-encontrados-e-como-resolvi) — deveria ser restrito aos ARNs específicos.
- **`Anywhere-IPv4`** na porta 3001 — em produção, a origem seria um Load Balancer ou uma faixa conhecida.

Documentar o que ainda não está ideal faz parte do trabalho: em segurança, o risco que se conhece é o que se pode planejar.

---

## 12. Créditos e referências

**Formação**

- **Mentoria Desafios Fundamentais — Formação AWS**, conduzida por **Henrylle Maia**
- Projeto BIA: [github.com/henrylle/bia](https://github.com/henrylle/bia)

**Documentação oficial consultada**

- [AWS CLI — Instalação (Linux)](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [AWS Systems Manager — Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Session Manager Plugin — Instalação](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)
- [IAM — Credenciais de segurança temporárias](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
- [Amazon ECR — Autenticação de registry privado](https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html)
- [Amazon EC2 — Security Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-security-groups.html)
- [WSL — Documentação Microsoft](https://learn.microsoft.com/pt-br/windows/wsl/)
- [Docker Desktop — WSL 2 backend](https://docs.docker.com/desktop/wsl/)

---

<div align="center">

**Rafael Silva** · [GitHub](https://github.com/RafaelSilva89)

*Desafio 2 — Formação AWS · Desafios Fundamentais*

</div>
