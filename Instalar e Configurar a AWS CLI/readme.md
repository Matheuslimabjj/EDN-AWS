Instalar e Configurar a AWS CLI

# Lab AWS - Instalar e Configurar a AWS CLI

## 📋 Sobre o Lab

Este laboratório faz parte do **Programa Re/Start AWS** através da **Escola da Nuvem**, focado em práticas de instalação, configuração e uso da AWS Command Line Interface (AWS CLI) em uma instância EC2 Linux.

## 🎯 Objetivos

Ao concluir este laboratório, pratiquei:

- ✅ Conectar a uma instância EC2 via SSH
- ✅ Instalar a AWS CLI em uma instância Amazon Linux
- ✅ Configurar a AWS CLI com credenciais de acesso (Access Key + Secret Key)
- ✅ Acessar o IAM usando comandos da AWS CLI
- ✅ Listar e recuperar políticas do IAM via linha de comando

## 🏗️ Arquitetura do Lab

![Diagrama da Arquitetura](./screenshots/arquitetura.png)

O laboratório simula um cenário em que o usuário acessa remotamente uma instância EC2 via SSH, utiliza a AWS CLI instalada na instância para se autenticar na conta AWS e interage com o serviço IAM programaticamente — sem depender do Console de Gerenciamento da AWS.

| Componente | Descrição |
|---|---|
| Usuário | Acesso local via terminal |
| Conexão SSH | Autenticação via par de chaves (.pem) |
| Instância EC2 | Amazon Linux 2 (ip-10-200-0-207) |
| AWS CLI | Ferramenta de linha de comando instalada na instância |
| AWS IAM | Serviço acessado via CLI para consulta de usuários e políticas |

## 🔧 Tecnologias e Serviços Utilizados

- **Amazon EC2** — Instância Linux para execução da AWS CLI
- **AWS CLI v2** — Interface de linha de comando para interação com a AWS
- **AWS IAM** — Serviço de gerenciamento de identidades consultado via CLI
- **SSH** — Protocolo de acesso remoto seguro à instância

## 📝 Etapas Realizadas

### Tarefa 1: Conectar à Instância EC2 via SSH

Acessei a instância EC2 remotamente usando o arquivo de chave `.pem` fornecido pelo laboratório. Após aceitar a autenticidade do host, a conexão foi estabelecida com sucesso.

![Conexão SSH à Instância EC2](./screenshots/ssh-conexao.png)
*Conexão SSH estabelecida com a instância Amazon Linux 2 (ec2-user@52.12.120.97)*

**Comandos utilizados:**
```bash
chmod 400 labsuser.pem
ssh -i labsuser.pem ec2-user@<ip-publico>
whoami   # confirma o usuário logado: ec2-user
```

> **Nota:** A instância estava rodando Amazon Linux 2, que possui a AWS CLI pré-instalada. O laboratório também cobre a instalação manual para instâncias Red Hat, onde a CLI não vem pré-instalada.

---

### Tarefa 2: Instalar a AWS CLI

Realizei o download, descompactação e instalação da AWS CLI v2 diretamente na instância via terminal.

![Download e Descompactação da AWS CLI](./screenshots/install-awscli.png)
*Download do pacote awscliv2.zip e extração dos arquivos de instalação*

![Verificação da Versão e Teste do Help](./screenshots/awscli-version-help.png)
*Confirmação da instalação: aws-cli/2.34.17 rodando em Python 3.14.3*

**Comandos utilizados:**
```bash
# Download do instalador
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Descompactação
unzip -u awscliv2.zip

# Instalação (requer sudo para permissão de escrita)
sudo ./aws/install

# Verificação
aws --version
# Saída: aws-cli/2.34.17 Python/3.14.3 Linux/4.14.355-280.714.amzn2.x86_64

# Teste do sistema de ajuda
aws help
```

---

### Tarefa 3: Observar as Configurações do IAM no Console

Antes de configurar a CLI, inspecionei o usuário `awsstudent` no Console do IAM para entender as permissões e localizar as credenciais de acesso.

![Permissões do Usuário awsstudent — lab_policy JSON](./screenshots/iam-permissoes-labpolicy.png)
*Política lab_policy em JSON anexada ao usuário awsstudent — lista de ações permitidas em EC2, CloudFormation, CloudWatch e outros serviços*

![Credenciais de Segurança do awsstudent](./screenshots/iam-credenciais.png)
*Aba "Credenciais de segurança" do usuário awsstudent com as duas chaves de acesso ativas*

**O que foi observado:**
- O usuário `awsstudent` possui a política `lab_policy` (gerenciada pelo cliente) com permissões específicas em EC2, CloudFormation, CloudWatch e S3
- Também possui a política `ReadOnlyAccess` (gerenciada pela AWS)
- Duas chaves de acesso ativas estavam disponíveis para uso com a CLI

---

### Tarefa 4: Configurar a AWS CLI

Com as credenciais em mãos, executei o comando `aws configure` para autenticar a CLI à conta AWS.

![Configuração aws configure e Listagem de Usuários IAM](./screenshots/aws-configure-list-users.png)
*Configuração da AWS CLI com Access Key, Secret Key, região us-west-2 e formato JSON — seguida do comando aws iam list-users retornando o usuário awsstudent*

**Comando e parâmetros configurados:**
```bash
aws configure
```

| Parâmetro | Valor configurado |
|---|---|
| AWS Access Key ID | AKIA5S7VN4JYFZ7VYNNN |
| AWS Secret Access Key | `****` (fornecida pelo lab) |
| Default region name | us-west-2 |
| Default output format | json |

> **Importante:** As chaves de acesso são o equivalente programático do usuário/senha do console. Nunca devem ser commitadas em repositórios públicos.

---

### Tarefa 5: Acessar o IAM via AWS CLI

Com a CLI configurada, executei comandos para listar usuários e interagir com o IAM diretamente pelo terminal.

```bash
aws iam list-users
```

**Resposta JSON retornada:**
```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "awsstudent",
            "UserId": "AIDA5S7VN4JYHPLSXQQX7",
            "Arn": "arn:aws:iam::934133228144:user/awsstudent",
            "CreateDate": "2026-03-26T02:21:39+00:00"
        }
    ]
}
```

---

### Desafio: Baixar a lab_policy via CLI

O desafio consistia em recuperar o documento JSON da `lab_policy` **usando apenas a AWS CLI**, sem recorrer ao Console de Gerenciamento da AWS.

![Desafio — list-policies e get-policy-version](./screenshots/desafio-list-policies.png)
*Execução do aws iam list-policies filtrando políticas locais, seguida da tentativa de recuperar a versão da política*

**Passo 1 — Listar políticas gerenciadas pelo cliente:**
```bash
aws iam list-policies --scope Local
```

Retornou os metadados da `lab_policy`, incluindo o ARN e o `DefaultVersionId`:
```json
{
    "PolicyName": "lab_policy",
    "PolicyId": "ANPA5S7VN4JYMVKFKGNIK",
    "Arn": "arn:aws:iam::934133228144:policy/lab_policy",
    "DefaultVersionId": "v1"
}
```

**Passo 2 — Recuperar o documento JSON da política e salvar em arquivo:**
```bash
aws iam get-policy-version \
  --policy-arn arn:aws:iam::934133228144:policy/lab_policy \
  --version-id v1 > lab_policy.json
```

> **Obs.:** O lab usou um ARN de exemplo diferente do ARN real da conta, o que gerou um erro `NoSuchEntity`. O comando correto usa o ARN retornado pelo `list-policies` da própria conta (`934133228144`), não o ARN do gabarito do lab (`038946776283`). O arquivo `lab_policy.json` foi gerado com sucesso após a correção.

**Lição aprendida:** Sempre use os valores retornados pelos próprios comandos da CLI em vez de copiar ARNs de exemplos estáticos — eles variam por conta e sessão de laboratório.

## 🔐 Conceitos-Chave Aprendidos

### Dois Modos de Acesso à AWS

| Modo | Credencial | Interface |
|---|---|---|
| Console de Gerenciamento | Usuário + Senha | Navegador web (GUI) |
| AWS CLI / SDK / API | Access Key + Secret Key | Terminal / Código |

Ambos os modos acessam **a mesma conta AWS** e são controlados pelas mesmas políticas do IAM — a diferença é apenas a interface de autenticação.

### Estrutura do aws configure

O comando `aws configure` cria dois arquivos locais na instância:

```
~/.aws/credentials   → Access Key ID + Secret Access Key
~/.aws/config        → Região padrão + formato de saída
```

Esses arquivos são lidos automaticamente pela CLI em todas as chamadas subsequentes.

### Fluxo de um Comando AWS CLI

```
Terminal → aws iam list-users
              ↓
        AWS CLI (autentica com as credenciais configuradas)
              ↓
        API do IAM na região us-west-2
              ↓
        Resposta JSON exibida no terminal
```

### Filtrando Políticas com --scope

```bash
aws iam list-policies --scope Local    # Apenas políticas gerenciadas pelo cliente
aws iam list-policies --scope AWS      # Apenas políticas gerenciadas pela AWS
aws iam list-policies                  # Todas as políticas (padrão: All)
```

## 💡 Principais Aprendizados

1. **CLI vs Console** — A AWS CLI oferece as mesmas capacidades do Console, mas via linha de comando. Isso é essencial para automação, scripts e pipelines de CI/CD.

2. **Credenciais são sensíveis** — Access Keys têm o mesmo poder que um login/senha. Devem ser rotacionadas periodicamente e nunca expostas em código ou repositórios públicos.

3. **ARNs são únicos por conta** — O Amazon Resource Name (ARN) de um recurso inclui o ID da conta AWS. Copiar ARNs de exemplos externos sem adaptá-los causa erros.

4. **`--scope Local` é fundamental** — Sem esse filtro, `list-policies` retorna milhares de políticas gerenciadas pela AWS, dificultando encontrar políticas customizadas.

5. **Redirecionamento `>`** — O operador `>` no Linux redireciona a saída do comando para um arquivo, o que é útil para salvar respostas JSON da CLI para uso posterior.

## 🚀 Como Reproduzir este Lab

### Pré-requisitos
- Acesso ao AWS Academy Lab
- Terminal com suporte a SSH (macOS/Linux nativo; Windows via PuTTY ou WSL)
- Arquivo `.pem` da instância EC2

### Resumo do Passo a Passo

```bash
# 1. Conectar via SSH
chmod 400 labsuser.pem
ssh -i labsuser.pem ec2-user@<ip-publico>

# 2. Instalar a AWS CLI (se não estiver pré-instalada)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip -u awscliv2.zip
sudo ./aws/install
aws --version

# 3. Configurar a CLI
aws configure
# → Inserir Access Key, Secret Key, região (us-west-2), formato (json)

# 4. Testar a conexão
aws iam list-users

# 5. Desafio: recuperar a lab_policy
aws iam list-policies --scope Local
aws iam get-policy-version \
  --policy-arn <arn-retornado-acima> \
  --version-id v1 > lab_policy.json
```

## 📊 Resultados

| Métrica | Valor |
|---|---|
| Instâncias EC2 acessadas | 1 |
| AWS CLI instalada e configurada | ✅ |
| Usuários IAM listados via CLI | 1 (awsstudent) |
| Políticas recuperadas via CLI | 1 (lab_policy) |
| Arquivo lab_policy.json gerado | ✅ |

## 📚 Recursos Adicionais

- [Documentação Oficial AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/)
- [Referência de Comandos IAM na AWS CLI](https://docs.aws.amazon.com/cli/latest/reference/iam/)
- [Instalação da AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Boas Práticas de Chaves de Acesso IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#lock-away-credentials)
- [AWS Academy](https://aws.amazon.com/training/awsacademy/)

## 🏆 Certificações Relacionadas

Este laboratório contribui para a preparação das seguintes certificações:

- **AWS Certified Cloud Practitioner**
- **AWS Certified Solutions Architect - Associate**
- **AWS Certified SysOps Administrator - Associate**

## 👨‍💻 Autor

**Matheus Lima**
Estudante — Escola da Nuvem | Programa Re/Start AWS

---

## 📄 Licença

Este projeto é parte do Programa Re/Start AWS e está disponível para fins de estudo e portfólio.

---

<div align="center">

[![AWS](https://img.shields.io/badge/AWS-Academy-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/training/awsacademy/)
[![CLI](https://img.shields.io/badge/AWS-CLI-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/cli/)
[![IAM](https://img.shields.io/badge/AWS-IAM-569A31?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/iam/)

</div>
