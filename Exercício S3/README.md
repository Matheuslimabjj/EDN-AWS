# Lab AWS — Amazon S3: Armazenamento de Objetos na Nuvem

## 📋 Sobre o Lab

Este laboratório faz parte do **Programa Re/Start AWS** através da **Escola da Nuvem**, focado em práticas fundamentais de armazenamento de objetos com Amazon S3, incluindo criação de buckets, upload de objetos, configuração de permissões públicas e interação via AWS CLI.

## 🎯 Objetivos

Ao concluir este laboratório, pratiquei:

- ✅ Criar um bucket S3 via AWS CLI
- ✅ Fazer upload de um objeto para o bucket
- ✅ Tentar acessar o objeto via browser (acesso negado — comportamento esperado)
- ✅ Tornar o objeto publicamente acessível via Bucket Policy
- ✅ Acessar o objeto publicamente via browser
- ✅ Listar o conteúdo do bucket via AWS CLI

## 🏗️ Arquitetura do Lab

### Infraestrutura Utilizada

| Componente | Detalhes |
|---|---|
| CLI Host | Amazon Linux 2 — EC2 — acesso via EC2 Instance Connect |
| S3 Bucket | `meu-bucket-lab-s3-1776050576` — região `us-west-2` |
| Objeto | `index.html` — 38 bytes — HTML estático |
| Acesso público | Bucket Policy com `s3:GetObject` para `Principal: *` |
| Region | `us-west-2` (Oregon) |

O fluxo do lab parte da instância CLI Host, já provisionada pelo ambiente do lab. A partir dela, via EC2 Instance Connect, a AWS CLI é usada para criar o bucket, fazer upload do objeto e configurar as permissões de acesso público.

```
Console AWS
    │
    └── EC2 Instance Connect ──► CLI Host (Amazon Linux 2)
                                        │
                                  AWS CLI configurada
                                  (aws configure)
                                        │
                         ┌──────────────┼──────────────┐
                         │              │               │
                   aws s3 mb      echo > index.html   aws s3api
                 (cria bucket)   (cria objeto local)  put-bucket-policy
                         │              │               │
                         └──────────────┘               │
                                  aws s3 cp             │
                               (upload objeto)          │
                                        │               │
                                 S3 Bucket ◄────────────┘
                                        │
                              Bucket Policy aplicada
                            (s3:GetObject para Principal: *)
                                        │
                    https://<bucket>.s3.us-west-2.amazonaws.com/index.html
                                        │
                                   Browser ✅
```

## 🔧 Tecnologias e Serviços Utilizados

- **Amazon S3** — Armazenamento de objetos escalável e durável na nuvem
- **AWS CLI** — Interface de linha de comando para interação com os serviços AWS
- **EC2 Instance Connect** — Acesso seguro à instância CLI Host sem par de chaves
- **S3 Bucket Policy** — Controle de acesso baseado em JSON para recursos do bucket
- **Amazon EC2** — Instância usada como ambiente de execução da CLI

## 📝 Etapas Realizadas

### Tarefa 1: Conectar ao CLI Host e Configurar a AWS CLI

O acesso foi feito via EC2 Instance Connect diretamente pelo Console AWS. Com o terminal aberto, a AWS CLI foi configurada com as credenciais do ambiente do lab.

![Terminal — aws configure e criação do bucket](./Screenshots/01-aws-configure-bucket-create.png)
*Configuração da AWS CLI com AccessKey, SecretKey e região `us-west-2`, seguida da criação do bucket com nome baseado em timestamp*

**Comandos executados:**

```bash
aws configure
```

Valores fornecidos nos prompts:
- **AWS Access Key ID:** `AKIASLQRIFRCWZVCYJHA`
- **AWS Secret Access Key:** `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`
- **Default region name:** `us-west-2`
- **Default output format:** `json`

---

### Tarefa 2: Criar o Bucket S3

Com a CLI configurada, o bucket foi criado com nome único baseado em timestamp Unix.

```bash
aws s3 mb s3://meu-bucket-lab-s3-$(date +%s) --region us-west-2
```

Saída retornada:
```
make_bucket: meu-bucket-lab-s3-1776050576
```

> **Nota:** O uso de `$(date +%s)` no nome garante unicidade global do bucket, já que nomes de buckets S3 são únicos em toda a AWS.

---

### Tarefa 3: Criar e Fazer Upload do Objeto

Um arquivo HTML simples foi criado localmente e enviado para o bucket.

**Criação do arquivo local:**

```bash
echo '<h1>Hello from S3 Challenge Lab!</h1>' > index.html
```

> **Atenção:** O uso de aspas simples (`'`) é obrigatório neste comando. Com aspas duplas, o bash interpreta o caractere `!` como um evento de histórico, causando o erro `-bash: !: event not found`.

**Upload para o S3:**

```bash
aws s3 cp index.html s3://meu-bucket-lab-s3-1776050576/ --region us-west-2
```

Saída retornada:
```
upload: ./index.html to s3://meu-bucket-lab-s3-1776050576/index.html
```

---

### Tarefa 4: Tentar Acessar o Objeto via Browser (Access Denied)

Com o objeto no bucket, foi feita a primeira tentativa de acesso via browser usando a URL pública do S3:

```
https://meu-bucket-lab-s3-1776050576.s3.us-west-2.amazonaws.com/index.html
```

![Browser — Access Denied](./Screenshots/02-browser-access-denied.png)
*Resposta XML com `AccessDenied` ao tentar acessar o objeto antes de configurar as permissões públicas — comportamento esperado e necessário para o lab*

O retorno foi o XML de erro esperado:
```xml
<Error>
  <Code>AccessDenied</Code>
  <Message>Access Denied</Message>
</Error>
```

> Este comportamento é o correto. Por padrão, todos os objetos S3 são privados. Este screenshot documenta o estado inicial antes de qualquer configuração de acesso público.

---

### Tarefa 5: Tornar o Objeto Publicamente Acessível

A abordagem inicial foi tentar usar ACLs de objeto (`put-object-acl`). Porém, como o bucket foi criado com **Object Ownership: Bucket owner enforced** (padrão desde abril de 2023), as ACLs são desabilitadas. A solução foi usar uma **Bucket Policy**.

![Terminal — erros de ACL e aplicação da Bucket Policy](./Screenshots/03-acl-errors-bucket-policy-applied.png)
*Sequência mostrando os erros de ACL (`AccessDenied` e `AccessControlListNotSupported`), a criação do `policy.json` via heredoc e a execução bem-sucedida do `put-bucket-policy` sem erros*

**Diagnóstico dos erros encontrados:**

| Tentativa | Erro | Causa Raiz |
|---|---|---|
| `put-object-acl` direto | `AccessDenied` — public ACLs are prevented by BlockPublicAcls | Block Public Access estava habilitado |
| `put-public-access-block` + `put-object-acl` | `AccessControlListNotSupported` — bucket does not allow ACLs | Object Ownership definido como `BucketOwnerEnforced` desabilita ACLs |
| `put-bucket-ownership-controls` | Erro de sintaxe — CLI versão antiga não suporta o parâmetro | Versão do AWS CLI na instância é anterior ao suporte ao comando |

**Solução aplicada — Bucket Policy:**

**Step 1 — Remover o bloqueio de acesso público:**
```bash
aws s3api put-public-access-block \
  --bucket meu-bucket-lab-s3-1776050576 \
  --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**Step 2 — Criar o arquivo de policy:**
```bash
cat > policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::meu-bucket-lab-s3-1776050576/*"
    }
  ]
}
EOF
```

**Step 3 — Aplicar a Bucket Policy:**
```bash
aws s3api put-bucket-policy \
  --bucket meu-bucket-lab-s3-1776050576 \
  --policy file://policy.json
```

Saída: *(nenhuma — ausência de erro é confirmação de sucesso na AWS CLI)*

---

### Tarefa 6: Acessar o Objeto Publicamente via Browser

Após aplicar a Bucket Policy, o mesmo URL foi acessado novamente no browser.

![Browser — Hello from S3 Challenge Lab!](./Screenshots/04-browser-public-access-success.png)
*Objeto `index.html` acessível publicamente via browser, exibindo o conteúdo HTML renderizado: "Hello from S3 Challenge Lab!"*

```
https://meu-bucket-lab-s3-1776050576.s3.us-west-2.amazonaws.com/index.html
```

O conteúdo HTML foi renderizado corretamente pelo browser, confirmando que o objeto está público.

---

### Tarefa 7: Listar o Conteúdo do Bucket via AWS CLI

A última etapa foi listar os objetos do bucket para confirmar o que está armazenado.

![Terminal — aws s3 ls com resultado](./Screenshots/05-s3-ls-final.png)
*Saída do comando `aws s3 ls` mostrando o objeto `index.html` com 38 bytes, criado em `2026-04-13 03:27:45`*

```bash
aws s3 ls s3://meu-bucket-lab-s3-1776050576/
```

Saída retornada:
```
2026-04-13 03:27:45         38 index.html
```

---

## 🔐 Conceitos-Chave Aprendidos

### S3 — Modelo de Acesso e Privacidade por Padrão

Por padrão, todo bucket e objeto S3 é privado. Nenhum acesso externo é permitido sem configuração explícita. Existem duas formas principais de conceder acesso público a objetos:

```
Opção 1 — ACL de Objeto (legada, desabilitada por padrão desde 2023):
  aws s3api put-object-acl --acl public-read
  ❌ Não funciona com Object Ownership: BucketOwnerEnforced

Opção 2 — Bucket Policy (recomendada):
  aws s3api put-bucket-policy --policy file://policy.json
  ✅ Funciona independente da configuração de Object Ownership
```

### Block Public Access — Camada de Proteção do Bucket

O S3 possui quatro configurações de Block Public Access que funcionam como uma camada extra de proteção, sobreposta às ACLs e Bucket Policies:

| Configuração | O que bloqueia |
|---|---|
| `BlockPublicAcls` | Impede criação de ACLs públicas |
| `IgnorePublicAcls` | Ignora ACLs públicas existentes |
| `BlockPublicPolicy` | Bloqueia aplicação de Bucket Policies públicas |
| `RestrictPublicBuckets` | Restringe acesso público mesmo com policy aplicada |

Todas precisam ser `false` para que uma Bucket Policy de acesso público seja efetiva.

### Object Ownership — Controle sobre ACLs

Desde abril de 2023, buckets novos são criados com **Object Ownership: Bucket owner enforced**, o que desabilita completamente o sistema de ACLs. Isso significa que `put-object-acl` retornará `AccessControlListNotSupported`. A solução é usar Bucket Policies.

```
Bucket criado antes de 2023:
  Object Ownership: Object Writer (padrão antigo)
  → ACLs funcionam normalmente

Bucket criado após abril de 2023:
  Object Ownership: Bucket owner enforced (novo padrão)
  → ACLs desabilitadas → usar Bucket Policy
```

### Bucket Policy — Controle de Acesso Baseado em Recurso

A Bucket Policy é um documento JSON anexado ao bucket que define quem pode fazer o quê com quais recursos. A policy aplicada neste lab concede acesso de leitura a qualquer pessoa (`Principal: *`) para todos os objetos do bucket (`Resource: arn:aws:s3:::bucket-name/*`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::meu-bucket-lab-s3-1776050576/*"
    }
  ]
}
```

> `s3:GetObject` permite leitura do conteúdo do objeto. Sem essa permissão, o browser recebe `AccessDenied`.

### Nomenclatura de Buckets S3

Nomes de buckets S3 são **globais** — únicos em toda a AWS, independente de conta ou região. Boas práticas:
- Usar lowercase (sem maiúsculas)
- Sem underscores — usar hífen
- Entre 3 e 63 caracteres
- Não pode parecer um IP (ex: `192.168.0.1`)

O lab usa `$(date +%s)` (timestamp Unix) para garantir unicidade automaticamente.

### URL Pública de um Objeto S3

A estrutura da URL pública de um objeto S3 segue o padrão:

```
https://<bucket-name>.s3.<region>.amazonaws.com/<object-key>

Exemplo:
https://meu-bucket-lab-s3-1776050576.s3.us-west-2.amazonaws.com/index.html
```

### Silêncio = Sucesso na AWS CLI

Comandos como `aws s3api put-bucket-policy` e `aws s3api put-public-access-block` não retornam saída em caso de sucesso. A ausência de mensagem de erro é a confirmação de que o comando foi executado corretamente.

## 💡 Principais Aprendizados

1. **Aspas simples vs. aspas duplas no bash** — O caractere `!` dentro de aspas duplas é interpretado pelo bash como expansão de histórico, causando `-bash: !: event not found`. Aspas simples desabilitam toda expansão do shell.

2. **ACLs S3 foram descontinuadas como padrão** — Buckets criados após abril de 2023 têm ACLs desabilitadas por padrão. Bucket Policies são o método moderno e recomendado para controle de acesso.

3. **Block Public Access tem precedência** — Mesmo com uma Bucket Policy de acesso público aplicada, o Block Public Access pode bloqueá-la. Ambos precisam estar alinhados.

4. **Depuração em camadas no S3** — Quando um objeto não é acessível, investigar nesta ordem: (1) Block Public Access → (2) Object Ownership / ACLs → (3) Bucket Policy → (4) IAM Policy.

5. **`$(date +%s)` para nomes únicos** — Uma técnica simples e eficaz para gerar nomes únicos de recursos sem depender de UUIDs ou variáveis externas.

6. **`file://` para passar JSON pela CLI** — O prefixo `file://` instrui a AWS CLI a ler o conteúdo do parâmetro de um arquivo local, em vez de receber o JSON inline, o que evita problemas de escaping de caracteres especiais no shell.

## 🚀 Como Reproduzir este Lab

### Pré-requisitos
- Acesso ao AWS Academy Lab
- Navegador web (Chrome, Firefox ou Edge)
- Conhecimento básico de terminal Linux e AWS CLI

### Resumo do Passo a Passo

1. **EC2 Instance Connect** — Conectar ao CLI Host pelo Console AWS
2. **`aws configure`** — Configurar credenciais, região `us-west-2` e formato `json`
3. **`aws s3 mb`** — Criar bucket com nome único via timestamp
4. **`echo + aws s3 cp`** — Criar o arquivo `index.html` e fazer upload
5. **Acessar via browser** — Confirmar o `AccessDenied` (screenshot obrigatório)
6. **`put-public-access-block`** — Desabilitar o bloqueio de acesso público
7. **Criar `policy.json`** — Definir a Bucket Policy via heredoc
8. **`put-bucket-policy`** — Aplicar a policy ao bucket
9. **Acessar via browser novamente** — Confirmar acesso público ao objeto (screenshot obrigatório)
10. **`aws s3 ls`** — Listar o conteúdo do bucket (screenshot obrigatório)

## 📊 Resultados

| Métrica | Valor |
|---|---|
| Bucket S3 criado | `meu-bucket-lab-s3-1776050576` — região `us-west-2` |
| Objeto enviado | `index.html` — 38 bytes |
| Método de acesso público | Bucket Policy (`s3:GetObject` para `Principal: *`) |
| ACLs funcionaram? | ❌ — Object Ownership: BucketOwnerEnforced |
| Problema com aspas duplas | `-bash: !: event not found` → resolvido com aspas simples |
| Objeto acessível via browser | ✅ |
| Listagem via CLI | ✅ — `2026-04-13 03:27:45  38 index.html` |

## 🗂️ Tabela de Renomeação das Screenshots

| Nome Original | Novo Nome | O que está sendo mostrado | Status |
|---|---|---|---|
| `Screenshot_from_2026-04-13_00-26-39.png` | `01-aws-configure-bucket-create.png` | Terminal — `aws configure` + criação do bucket + erro com aspas duplas no `echo` | ✅ Relevante |
| `Screenshot_from_2026-04-13_00-26-49.png` | — | Credentials panel com SecretKey, PublicIP e AccessKey | ⚠️ Contexto — não referenciado no README (credenciais visíveis) |
| `Screenshot_from_2026-04-13_00-28-10.png` | `02-browser-access-denied.png` | Browser — `AccessDenied` XML na 1ª tentativa de acesso ao objeto | ✅ Relevante |
| `Screenshot_from_2026-04-13_00-29-19.png` | — | Terminal — erro ACL `AccessDenied` + execução do `put-public-access-block` | ⚠️ Intermediário — coberto pelo screenshot 03 |
| `Screenshot_from_2026-04-13_00-29-44.png` | — | Terminal — erro `AccessControlListNotSupported` (bucket does not allow ACLs) | ⚠️ Intermediário — coberto pelo screenshot 03 |
| `Screenshot_from_2026-04-13_00-30-09.png` | — | Terminal — tentativa falha de `put-bucket-ownership-controls` (CLI antiga) | ⚠️ Intermediário — coberto pelo screenshot 03 |
| `Screenshot_from_2026-04-13_00-30-47.png` | — | Terminal — tentativa com JSON também falha, ainda `AccessControlListNotSupported` | ⚠️ Intermediário — coberto pelo screenshot 03 |
| `Screenshot_from_2026-04-13_00-31-19.png` | `03-acl-errors-bucket-policy-applied.png` | Terminal — `cat policy.json` via heredoc + `put-bucket-policy` executado com sucesso + `aws s3 ls` | ✅ Relevante |
| `Screenshot_from_2026-04-13_00-32-02.png` | `04-browser-public-access-success.png` | Browser — **Hello from S3 Challenge Lab!** renderizado com sucesso | ✅ Relevante |
| `Screenshot_from_2026-04-13_00-32-12.png` | — | Terminal — mesmo conteúdo do screenshot 03 (duplicata) | ❌ Duplicata — descartar |
| `Screenshot_from_2026-04-13_00-32-34.png` | `05-s3-ls-final.png` | Terminal — `aws s3 ls` com resultado final mostrando `index.html` 38 bytes | ✅ Relevante |

> **Screenshots mínimas para o lab:** `01`, `02`, `03`, `04`, `05` — as demais são intermediárias ou duplicatas e podem ser descartadas sem prejuízo para a documentação.

## 📚 Recursos Adicionais

- [Documentação Amazon S3](https://docs.aws.amazon.com/s3/)
- [AWS CLI — Referência S3](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3/index.html)
- [AWS CLI — Referência S3API](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/s3api/index.html)
- [Usando Bucket Policies para Acesso Público](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-policy-language-overview.html)
- [Block Public Access — Documentação](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html)
- [Object Ownership — Documentação](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)
- [AWS Academy](https://aws.amazon.com/training/awsacademy/)

## 🏆 Certificações Relacionadas

Este laboratório contribui para a preparação das seguintes certificações:

- **AWS Certified Cloud Practitioner**
- **AWS Certified Solutions Architect - Associate**
- **AWS Certified Developer - Associate**

## 👨‍💻 Autor

**Matheus Lima**

Estudante — Escola da Nuvem | Programa Re/Start AWS

---

## 📄 Licença

Este projeto é parte do Programa Re/Start AWS e está disponível para fins de estudo e portfólio.

---

<div align="center">

[![AWS](https://img.shields.io/badge/AWS-Academy-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/training/awsacademy/)
[![S3](https://img.shields.io/badge/Amazon-S3-569A31?style=for-the-badge&logo=amazon-s3&logoColor=white)](https://aws.amazon.com/s3/)
[![CLI](https://img.shields.io/badge/AWS-CLI-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/cli/)
[![EC2](https://img.shields.io/badge/Amazon-EC2-FF9900?style=for-the-badge&logo=amazon-ec2&logoColor=white)](https://aws.amazon.com/ec2/)

</div>
