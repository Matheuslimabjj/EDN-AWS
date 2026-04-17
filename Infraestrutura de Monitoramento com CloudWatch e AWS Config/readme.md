# Lab AWS - Infraestrutura de Monitoramento com CloudWatch e AWS Config

## 📋 Sobre o Lab

Este laboratório, parte do programa **AWS re/Start** através da **Escola da Nuvem**, foca em um dos pilares mais críticos da nuvem: **monitoramento e observabilidade**. Aprendi a configurar uma stack completa para coletar métricas, logs e eventos, além de auditar a conformidade da infraestrutura.

## 🎯 Objetivos

Ao concluir este laboratório, pratiquei:

- ✅ Instalar e configurar o **CloudWatch Agent** via AWS Systems Manager
- ✅ Coletar **logs de aplicação** (Apache) e **métricas do sistema** (CPU, Memória, Disco)
- ✅ Criar **filtros de métrica** para detectar erros específicos (HTTP 404) em logs
- ✅ Configurar **alarmes do CloudWatch** e notificações por e-mail (SNS)
- ✅ Criar **regras no EventBridge** para reagir a mudanças de estado de instâncias EC2
- ✅ Auditar a **conformidade da infraestrutura** usando o AWS Config (tags e volumes órfãos)

## 🏗️ Arquitetura da Solução de Monitoramento

### Tarefa 1: Coleta de Logs e Métricas
![Arquitetura de Coleta de Dados](./screenshots/01-diagrama-coleta.png)
*Fluxo: Systems Manager instala o Agente -> EC2 -> Parameter Store (configuração) -> CloudWatch Logs e Métricas*

### Tarefa 2: Alerta Baseado em Logs
![Arquitetura de Alerta](./screenshots/02-diagrama-alerta.png)
*Fluxo: EC2 (Apache Logs) -> CloudWatch Logs -> Filtro de Métrica (HTTP 404) -> Alarme -> Notificação por E-mail (SNS)*

## 🔧 Tecnologias e Serviços Utilizados

- **AWS Systems Manager (Run Command)** - Automação da instalação do agente
- **Amazon CloudWatch (Agent, Logs, Metrics, Alarms)** - Coleta, armazenamento e alertas
- **Amazon SNS** - Notificações por e-mail
- **Amazon EventBridge** - Reação a eventos da infraestrutura
- **AWS Config** - Auditoria e conformidade de recursos

## 📝 Etapas Realizadas

### Tarefa 1: Instalar e Configurar o CloudWatch Agent

Usei o **AWS Systems Manager Run Command** para instalar o agente e, em seguida, configurá-lo via **Parameter Store**.

**1. Instalação do Agente:**
Utilizei o documento `AWS-ConfigureAWSPackage` para instalar o `AmazonCloudWatchAgent`.

![Instalação do Agente via SSM](./screenshots/03-ssm-install-success.png)
*Comando executado com sucesso na instância do Web Server*

**2. Configuração do Agente (Parameter Store):**
Criei um parâmetro chamado `Monitor-Web-Server` com a configuração JSON que define:
- **Logs a coletar:** `/var/log/httpd/access_log` e `error_log`
- **Métricas do sistema:** `cpu_usage_idle`, `mem_used_percent`, `used_percent` (disco), etc.

**3. Iniciar o Agente com a Configuração:**
Executei outro comando do SSM (`AmazonCloudWatch-ManageAgent`) apontando para a configuração no Parameter Store.

### Tarefa 2: Monitorar Logs e Criar Alertas (HTTP 404)

Com o agente enviando os logs do Apache para o **CloudWatch Logs**, criei um monitoramento para erros HTTP 404.

**1. Verificar os Logs no CloudWatch:**
Os logs do Apache chegaram no grupo de logs `HttpAccessLog`.

![Grupo de Logs HttpAccessLog no CloudWatch](./screenshots/04-cloudwatch-log-group.png)
*Log group criado automaticamente pelo agente, com o stream de log da instância*

**2. Visualizar os Eventos de Log:**
Dentro do stream, é possível ver as requisições GET e os erros 404.

![Eventos de log mostrando requisições e erro 404](./screenshots/05-log-events-404.png)
*Logs brutos do Apache, incluindo uma requisição para `/start` que resultou em erro 404*

**3. Criar um Filtro de Métrica:**
Criei um filtro com o padrão `[ip, id, user, timestamp, request, status_code=404, size]` para contar apenas os eventos com código 404.

![Criação do filtro de métrica para HTTP 404](./screenshots/06-create-metric-filter.png)
*Definição do padrão de filtro e teste com dados de log existentes*

**4. Configurar o Alarme e Notificação:**
Com base no filtro, criei um alarme que dispara se houver **5 ou mais erros 404 em 1 minuto**. A ação é enviar uma notificação para um tópico do **SNS** (e-mail).

![E-mail de Alarme do CloudWatch](./screenshots/07-alarm-email.png)
*Notificação recebida por e-mail confirmando que o alarme entrou em estado `ALARM`*

![Alarme no Console do CloudWatch](./screenshots/08-alarm-console.png)
*Console do CloudWatch mostrando o alarme "404 Errors" em estado vermelho (ALARM)*

### Tarefa 3: Monitorar Métricas Customizadas (CWAgent)

Além dos logs, o agente coleta métricas de dentro da instância, como uso de memória e espaço em disco.

![Métricas do CWAgent na instância EC2](./screenshots/09-cwagent-metrics.png)
*Visão das métricas de CPU, memória e disco coletadas pelo agente e exibidas no console do EC2*

### Tarefa 4: Criar Notificações em Tempo Real com EventBridge

Configurei o **Amazon EventBridge** para monitorar mudanças de estado das instâncias EC2 e enviar uma notificação quando uma instância for `stopped` ou `terminated`.

![Criando regra no EventBridge](./screenshots/10-eventbridge-rule.png)
*Regra configurada para capturar eventos "EC2 Instance State-change Notification" dos estados "stopped" e "terminated"*

![Instância EC2 sendo interrompida](./screenshots/11-ec2-stop-instance.png)
*Ação de interromper a instância, que disparará o evento para o EventBridge*

### Tarefa 5: Auditar Conformidade com AWS Config

Ativei o **AWS Config** e criei regras gerenciadas para verificar a conformidade da infraestrutura.

**1. Regra `required-tags`:**
Esta regra verifica se os recursos possuem a tag `project`. O Web Server estava em conformidade, mas outros recursos (como a VPC e sub-redes) não.

![Regra required-tags com status NON_COMPLIANT](./screenshots/12-config-required-tags.png)
*Resultado da auditoria mostrando que, embora a EC2 esteja OK, vários outros recursos não possuem a tag obrigatória*

**2. Regra `ec2-volume-inuse-check`:**
Esta regra verifica se volumes EBS estão anexados a alguma instância. O volume principal estava OK, mas um volume não utilizado foi identificado como não compatível.

![Regra ec2-volume-inuse-check com status COMPLIANT](./screenshots/13-config-volume-inuse.png)
*Auditoria mostrando um volume EBS em uso (compatível) e um volume órfão (não compatível - não visível neste print, mas o lab descreve a situação)*

## 💡 Principais Aprendizados

1.  **Centralização de Logs e Métricas:** O CloudWatch Logs elimina a necessidade de acessar cada servivo para debugar logs. Tudo fica em um local só.
2.  **Alertas Proativos:** Filtros de métrica em logs permitem criar alertas para problemas de aplicação (ex: muitos 404s, erros de banco de dados) sem mudar uma linha de código.
3.  **Monitoramento Holístico:** Métricas padrão da AWS (CPU, rede) + Métricas customizadas do agente (memória, disco) + Logs + Eventos (EventBridge) fornecem uma visão completa da saúde da aplicação.
4.  **Automação de Ações:** O EventBridge permite reagir automaticamente a eventos (ex: iniciar uma instância automaticamente, ou, como no lab, apenas notificar).
5.  **Governança e Compliance:** O AWS Config é essencial para garantir que sua esteja seguindo os padrões da empresa (tags) e as melhores práticas (sem recursos órfãos).

## 🚀 Como Reproduzir este Lab

### Pré-requisitos
- Uma instância EC2 com o Apache rodando (similar ao lab anterior)
- Permissões para criar regras no IAM (o lab fornece um ambiente preparado)

### Passo a Passo Resumido

1.  **Instalar o Agente:** Systems Manager -> Run Command -> `AWS-ConfigureAWSPackage` (Ação: Install, Nome: AmazonCloudWatchAgent).
2.  **Criar Configuração:** Systems Manager -> Parameter Store -> Criar parâmetro com o JSON do lab.
3.  **Iniciar Agente:** Run Command -> `AmazonCloudWatch-ManageAgent` (Ação: configure, Modo: ec2, Config Source: ssm, Config Location: `Monitor-Web-Server`).
4.  **Criar Filtro de Métrica:** CloudWatch -> Logs -> Log Group `HttpAccessLog` -> Ações -> Criar filtro de métrica.
    - Padrão: `[ip, id, user, timestamp, request, status_code=404, size]`.
5.  **Criar Alarme:** No filtro criado -> "Create Alarm" -> Condição: `... >= 5`, Ação: Criar/Usar tópico SNS.
6.  **Criar Regra EventBridge:** EventBridge -> Regras -> Criar regra.
    - Event Source: EC2 -> `EC2 Instance State-change Notification`.
    - States: `stopped`, `terminated`.
    - Target: SNS Topic (o mesmo usado antes).
7.  **Ativar AWS Config:** (Se não estiver ativo) -> Serviço Config -> Configurar.
8.  **Adicionar Regras:** AWS Config -> Regras -> Adicionar regra -> Buscar por `required-tags` e `ec2-volume-inuse-check`.

## 📚 Recursos Adicionais

- [Documentação do Amazon CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html)
- [Criar alarmes do CloudWatch baseados em filtros de métrica de logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)
- [Tutoriais do Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-get-started.html)
- [Regras Gerenciadas do AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)

## 👨‍💻 Autor

**Matheus Lima**  
Estudante - Escola da Nuvem

---

## 📄 Licença

Este projeto é parte do programa educacional AWS re/Start e está disponível para fins de estudo e portfólio.

<div align="center">

[![AWS](https://img.shields.io/badge/AWS-reStart-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/training/restart/)
[![CloudWatch](https://img.shields.io/badge/Amazon-CloudWatch-FF4F8B?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/cloudwatch/)
[![AWS Config](https://img.shields.io/badge/Amazon-Config-232F3E?style=for-the-badge&logo=amazon-aws&logoColor=white)](https://aws.amazon.com/config/)

</div>
