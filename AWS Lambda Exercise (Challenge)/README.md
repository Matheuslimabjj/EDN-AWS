# ☁️ AWS Lambda — Word Count Challenge

> Lab prático de computação serverless utilizando AWS Lambda, Amazon S3 e Amazon SNS.

---

## 📋 Descrição

Neste lab, foi desenvolvida uma função **AWS Lambda** em Python que conta automaticamente o número de palavras em arquivos de texto. A função é acionada sempre que um arquivo `.txt` é enviado para um bucket do **Amazon S3**, e o resultado é enviado por e-mail via **Amazon SNS**.

---

## 🏗️ Arquitetura

```
┌─────────────┐   upload .txt   ┌────────────────────┐   publica   ┌─────────────────┐   e-mail   ┌──────────┐
│  Amazon S3  │ ──────────────► │   AWS Lambda       │ ──────────► │   Amazon SNS    │ ─────────► │  Gmail   │
│   Bucket    │   (trigger)     │  WordCountFunction  │             │  WordCountTopic │            │          │
└─────────────┘                 └────────────────────┘             └─────────────────┘            └──────────┘
```

---

## 🎯 Objetivos

- [x] Criar uma função Lambda em Python para contar palavras em arquivos de texto
- [x] Configurar um bucket S3 para acionar a Lambda automaticamente no upload de `.txt`
- [x] Criar um tópico SNS para enviar o resultado por e-mail
- [x] Testar o pipeline completo com múltiplos arquivos

---

## 🛠️ Serviços AWS Utilizados

| Serviço | Função |
|---|---|
| **AWS Lambda** | Executa o código de contagem de palavras |
| **Amazon S3** | Armazena os arquivos `.txt` e aciona a Lambda |
| **Amazon SNS** | Envia o resultado por e-mail |
| **IAM** | Gerencia permissões via role `LambdaAccessRole` |
| **CloudWatch** | Registra logs de execução da função |

---

## 🐍 Código da Função Lambda

```python
import json
import boto3
import urllib.parse

def lambda_handler(event, context):
    s3_client = boto3.client('s3')
    sns_client = boto3.client('sns')

    SNS_TOPIC_ARN = 'arn:aws:sns:us-west-2:038896253995:WordCountTopic'

    # Obter bucket e arquivo do evento S3
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    file_key = urllib.parse.unquote_plus(
        event['Records'][0]['s3']['object']['key']
    )

    print(f"Bucket: {bucket_name} | File: {file_key}")

    # Ler o arquivo do S3
    response = s3_client.get_object(Bucket=bucket_name, Key=file_key)
    file_content = response['Body'].read().decode('utf-8')

    # Contar palavras
    word_count = len(file_content.split())

    # Formatar mensagem no padrão exigido
    message = f"The word count in the {file_key} file is {word_count}."
    subject = "Word Count Result"

    print(f"Result: {message}")

    # Publicar no SNS
    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Message=message,
        Subject=subject
    )

    return {
        'statusCode': 200,
        'body': json.dumps(message)
    }
```

---

## 🚀 Passo a Passo da Implementação

### 1. Criação do Tópico SNS
- Tipo: **Padrão (Standard)**
- Nome: `WordCountTopic`
- Assinatura por e-mail confirmada ✅

### 2. Criação do Bucket S3
- Nome: `word-count-bucket-6612`
- Região: `us-west-2`

### 3. Criação da Função Lambda
- Runtime: **Python 3.12**
- Arquitetura: **x86_64**
- Role de execução: **LambdaAccessRole**
  - `AWSLambdaBasicExecutionRole`
  - `AmazonSNSFullAccess`
  - `AmazonS3FullAccess`
  - `CloudWatchFullAccess`

![Função Lambda com código deployado](screenshots/lambda_function_code.png)

### 4. Configuração do Trigger S3
- Bucket: `word-count-bucket-6612`
- Evento: `PUT`
- Sufixo: `.txt`

![Trigger S3 configurado na Lambda](screenshots/lambda_trigger_s3.png)

### 5. Upload e Teste

Upload do arquivo `text1.txt` realizado com sucesso para o bucket S3:

![Upload bem-sucedido no S3](screenshots/s3_upload_success.png)

---

## ✅ Resultado Final

Após o upload, a Lambda foi invocada automaticamente e o e-mail foi recebido:

![E-mail com resultado da contagem de palavras](screenshots/email_word_count_result.png)

```
The word count in the text2.txt file is 24.
```

**Assunto:** Word Count Result  
**Remetente:** AWS Notifications `<no-reply@sns.amazonaws.com>`

---

## 📁 Estrutura do Repositório

```
aws-lambda-word-count/
├── README.md
├── lambda_function.py
└── screenshots/
    ├── lambda_function_code.png
    ├── lambda_trigger_s3.png
    ├── s3_upload_success.png
    └── email_word_count_result.png
```

---

## 📌 Formato da Mensagem

Conforme exigido pelo lab, a mensagem segue o padrão:

```
The word count in the <textFileName> file is nnn.
```

---

## 🌐 Região AWS

Todos os recursos foram criados na região **`us-west-2` (Oregon)**.

---

## 📚 Referências

- [O que é AWS Lambda?](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
- [Usar um trigger do S3 para invocar uma função Lambda](https://docs.aws.amazon.com/lambda/latest/dg/with-s3-example.html)
- [AWS Managed Policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_managed-vs-inline.html)

---

*Lab realizado como parte do programa AWS Training and Certification — 2026*
