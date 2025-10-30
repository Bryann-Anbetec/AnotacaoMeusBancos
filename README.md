
```markdown
---

## 🎯 **Visão Geral do Sistema "Meus Bancos"**

O **"Meus Bancos"** é uma **plataforma de gestão financeira corporativa** que permite:

- Integração com **múltiplos bancos brasileiros**
- Sincronização de **dados bancários** (extratos, saldos, boletos)
- Processamento de **pagamentos, tributos e conciliação**
- Integração com **sistemas ERP** (Winthor, Protheus, Sankya)
- Autenticação e autorização via **AWS Cognito**
- Processamento assíncrono via **Amazon SQS**

---

## 🧩 **Papel de Cada Projeto no Ecossistema**

### 1. **AB Server MyBanks**  
🔗 **Conector com Bancos**  
- Fornece uma **API unificada** para comunicação com bancos (BB, Santander, Bradesco, Itaú, Sicoob)
- Implementa o **padrão Strategy** para diferentes bancos
- Expõe endpoints para:
  - Boletos (`/bankslip`)
  - Extratos (`/statement`)
  - Pagamentos em lote (`/batch-payment`)
  - Tributos (`/tax-revenues`)
- Suporta **sandbox** via header `x-is-sandbox`

### 2. **AB Worker MyBanks**  
⚙️ **Orquestrador de Processos**  
- Gerencia **jobs assíncronos** (SQS)
- Processa:
  - Sincronização de saldos e extratos
  - Faturamento (`BillingJob`)
  - Pagamentos de tributos (`TributePaymentService`)
  - Integração com ERP (`ErpIntegrationService`)
- Mantém **estrutura de empresas, usuários e permissões**
- Autenticação via **AWS Cognito** (dois pools: admin e portal)

### 3. **AB Connector MyBanks**  
🔄 **Conector com ERPs**  
- Conecta sistemas ERP (**Winthor, Protheus, Sankya**) com a plataforma MyBanks
- Sincroniza dados em **ordem específica**:
  1. `CLIENTE`
  2. `NOTA_FISCAL`
  3. `CONTAS_RECEBER`
- Divide dados em **chunks** para processamento
- Usa **SQS FIFO** para garantir ordem e evitar duplicação

---

## 💼 **Principais Regras de Negócio**

### 1. **Autenticação e Autorização**
- Dois tipos de usuários:
  - **Portal Users**: Usuários empresariais (empresas)
  - **Admin Users**: Usuários administrativos (Anbetec)
- **MFA (Multi-Factor Authentication)** opcional com aprovação administrativa
- Perfis de acesso granulares por módulo e ação

### 2. **Gestão de Empresas**
- Empresas podem ter **filiais** (hierarquia)
- Cada empresa tem:
  - Configuração de **ERP** (tipo, credenciais, endpoints)
  - **Contas bancárias** vinculadas
  - **Usuários** com perfis específicos
- Status de integração com ERP controlado por `erpIntegrationActive`

### 3. **Sincronização Bancária**
- **Saldos diários** sincronizados automaticamente (Seg-Sáb, 6h)
- **Extratos** processados em tempo real
- **Conciliação automática** entre transações bancárias e lançamentos contábeis

### 4. **Processamento de Pagamentos**
- **Pagamentos em lote** com validação
- **Tributos** (ISS, ICMS, PIS) calculados automaticamente
- **DARF** gerado para pagamentos fiscais
- **Retorno de pagamentos** processado automaticamente

### 5. **Integração com ERP**
- **Ordem de processamento** obrigatória:
  1. Clientes
  2. Notas Fiscais  
  3. Contas a Receber
- **Chunking** de dados para performance:
  - Notas Fiscais: 10 registros/chunk
  - Outros: 100 registros/chunk
- **Teste de conexão** antes de iniciar integração

### 6. **Jobs e Processamento Assíncrono**
- **SQS FIFO** para garantir ordem e evitar duplicação
- **Status de jobs** monitorados:
  - `PENDING` → `IN_PROGRESS` → `COMPLETED`/`FAILED`/`PARTIALLY_COMPLETED`
- **Retry automático** com tratamento de falhas

### 7. **Conciliação Financeira**
- **Regras automáticas** para matching de transações
- **Nível de confiança** (0-100%) para conciliações
- **Conciliação manual** disponível para casos complexos

### 8. **Notificações**
- **Email** para:
  - Boas-vindas de usuários
  - Confirmação de pagamentos
  - Alertas do sistema
- **Regras de notificação** configuráveis por empresa

### 9. **Auditoria e Compliance**
- **Logs detalhados** de todas as operações
- **Rastreabilidade** completa (quem, quando, o que)
- **Backup** automático de dados críticos

---

## 🔄 **Fluxos de Negócio Principais**

### 1. **Integração Completa Empresa Nova**
```

1.  Criar empresa no sistema
2.  Configurar conexão com ERP
3.  Vincular contas bancárias
4.  Criar usuários e perfis
5.  Iniciar sincronização de dados
6.  Configurar regras de conciliação

<!-- end list -->

```

### 2. **Processamento de Pagamento**
```

1.  Lançamento no ERP → Connector
2.  Worker processa agendamento
3.  Server envia para banco
4.  Retorno processado automaticamente
5.  Status atualizado no ERP

<!-- end list -->

```

### 3. **Conciliação Diária**
```

1.  Sincronização automática de extratos (6h)
2.  Matching com lançamentos do ERP
3.  Conciliação automática (regras configuradas)
4.  Relatório de exceções para análise manual

<!-- end list -->

```

---

## 🛡️ **Regras de Segurança**

- **Certificados digitais** para APIs bancárias (AWS Secrets Manager)
- **Tokens OAuth2** para autenticação com bancos
- **SSL/TLS** em todas as conexões
- **Timeout** configurável para operações
- **Rate limiting** para APIs externas

---

## 📊 **Monitoramento e Qualidade**

- **Health checks** para todos os serviços
- **Logs estruturados** para debugging
- **Métricas de performance** (tempo de resposta, throughput)
- **Alertas** para falhas de integração
- **Dashboard** de status em tempo real (WebSockets)

---

## 🚀 **Fluxo de Dados entre os Projetos**

```

ERP → Connector → SQS → Worker → Server → Bancos
                ↆ       ↆ        ↆ
              Logs   Process  Response

```

1. **Connector** extrai dados do ERP e envia para **SQS**
2. **Worker** consome da SQS, processa e chama **Server** para operações bancárias
3. **Server** comunica com bancos e retorna respostas
4. **Worker** atualiza status e notifica sistemas

---

## 💡 **Diferenciais de Negócio**

1. **API Unificada**: Interface única para múltiplos bancos
2. **Orquestração Inteligente**: Processamento em ordem correta
3. **Conciliação Automática**: Matching inteligente de transações
4. **Escalabilidade**: Arquitetura baseada em eventos (SQS)
5. **Segurança Enterprise**: AWS Cognito + Certificados digitais
6. **Flexibilidade ERP**: Múltiplos ERPs suportados
```

