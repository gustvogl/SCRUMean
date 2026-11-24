# 🏗️ Definição da Arquitetura Tecnológica — Industrial Control Tower

**Versão:** 1.0  
**Status:** Especificação Técnica  
**Projeto:** Industrial Control Tower (Ambiente Industrial Simulado)

---

## 1. Visão Geral e Estilo Arquitetural

A arquitetura do **Industrial Control Tower** adota uma abordagem **modular em camadas desacopladas (N-Tier)** combinada com uma **arquitetura orientada a serviços internos**. Essa estrutura separa claramente a interface de apresentação, a orquestração transacional, o processamento analítico/inteligência operacional e a camada de persistência relacional.

### Objetivos Arquiteturais (Alinhados aos RNFs)
* **Desacoplamento e Modularidade (RNF06):** Permitir que a equipe desenvolva, teste e evolua o frontend, a API transacional e o motor de inteligência de forma independente.
* **Evolução Tecnológica (RNF08):** Garantir que o MVP inicie com regras de negócio determinísticas e evolua para modelos de *Machine Learning*, *Digital Twin* e integrações reais (ERP/MES) sem reescrever o núcleo do sistema.
* **Integridade e Rastreabilidade (RNF03, RNF04):** Assegurar consistência ponta a ponta entre cliente, pedido, matéria-prima, ordem de produção e entrega.
* **Segurança e Governança (RNF02, RNF07):** Controlar o acesso às informações conforme o perfil operacional de cada usuário.

---

## 2. Diagrama de Arquitetura em Camadas

```text
+-----------------------------------------------------------------------------------+
|                        CAMADA DE APRESENTAÇÃO (Frontend)                          |
|  React + CSS | Dashboards Executivos | Central de Alertas | Telas Departamentais  |
+-----------------------------------------+-----------------------------------------+
                                          | HTTPS / JSON (REST API + JWT)
                                          ▼
+-----------------------------------------------------------------------------------+
|                   CAMADA DE INTEGRAÇÃO E NEGÓCIO (Backend API)                    |
|  Node.js + Express | Autenticação (RBAC) | Validação | Parser Excel/CSV | Rotas   |
+-------------------+---------------------------------------------+-----------------+
                    | SQL / ORM                                   | HTTP Interno (JSON)
                    ▼                                             ▼
+---------------------------------------+   +---------------------------------------+
|    CAMADA DE PERSISTÊNCIA (Banco)     |   |   MOTOR DE INTELIGÊNCIA (Analytics)   |
|  PostgreSQL                           |◄──|  Python (FastAPI/Flask)               |
|  - Dados Transacionais e Mestres      |   |  - Pandas (Tratamento/Cruzamento)     |
|  - Histórico Operacional e Alertas    |   |  - Scikit-learn (Análise/Previsão)    |
+---------------------------------------+   +---------------------------------------+
```

---

## 3. Especificação das Camadas e Tecnologias

### 3.1. Camada de Apresentação (Frontend)
* **Tecnologias:** React, JavaScript/TypeScript, CSS moderno (Flexbox/Grid).
* **Responsabilidades:**
  * Renderizar o Dashboard Executivo com indicadores em tempo quase real e gráficos interativos.
  * Fornecer interfaces departamentais (Pedidos, Estoque, Produção, Logística) responsivas (RNF01).
  * Exibir a Central de Alertas com filtros por criticidade e fluxos de resolução.
  * Gerenciar o estado da sessão do usuário e adaptar menus/ações conforme o perfil autenticado.

### 3.2. Camada de Integração e Regras de Negócio (Backend)
* **Tecnologias:** Node.js com framework Express.
* **Responsabilidades:**
  * Expor APIs RESTful padronizadas para consumo do frontend.
  * Gerenciar autenticação baseada em tokens (**JWT**) e autorização por papéis (**RBAC**).
  * Realizar validação rigorosa dos payloads de entrada (RNF04) e tratamento amigável de erros (RNF05).
  * Processar a importação de planilhas **Excel (.xlsx) e CSV**, validando colunas e convertendo registros para o modelo relacional.
  * Orquestrar chamadas ao Motor de Inteligência sempre que houver alteração relevante na operação (ex.: novo pedido, baixa de estoque, alteração de ordem de produção).

### 3.3. Motor de Inteligência Operacional (Analytics & IA)
* **Tecnologias:** Python 3.10+, Pandas, Scikit-learn (exposto via micro-API interna com FastAPI ou Flask).
* **Responsabilidades:**
  * **Cruzamento Interdepartamental:** Relacionar saldo de matérias-primas, consumo previsto das ordens de produção, capacidade das linhas e prazos dos pedidos.
  * **Detecção de Riscos e Gargalos:**
    * *Risco de Atraso:* Comparar data de entrega do pedido com o lead time e status da ordem de produção.
    * *Risco de Ruptura de Estoque:* Confrontar estoque atual + compras previstas contra a demanda das ordens programadas.
    * *Gargalo Produtivo:* Calcular a taxa de ocupação ($\frac{\text{Demanda Programada}}{\text{Capacidade Disponível}}$) por linha de produção.
  * **Análise de Impacto em Cascata:** Mapear quais pedidos e entregas serão afetados caso ocorra falha em um insumo ou linha.
  * **Geração de Recomendações:** Sugerir ações corretivas (ex.: realocação de linha, priorização de ordem, solicitação emergencial de compra).

### 3.4. Camada de Persistência (Banco de Dados)
* **Tecnologia:** PostgreSQL.
* **Responsabilidades:**
  * Armazenar dados cadastrais, transacionais e históricos com integridade referencial garantida por chaves primárias e estrangeiras (RNF03).
  * Manter a rastreabilidade completa da cadeia: `Cliente -> Pedido -> Item -> Produto -> Matéria-Prima / Ordem de Produção -> Linha -> Entrega`.
  * Registrar o histórico de movimentações de estoque, ocorrências operacionais e ciclo de vida dos alertas (abertura, análise, resolução).

---

## 4. Fluxos de Comunicação e Integração

### 4.1. Fluxo Transacional e Analítico (Operação Normal)
1. O usuário cadastra um pedido ou importa uma planilha CSV/Excel no **Frontend (React)**.
2. O **Backend (Node.js/Express)** valida os dados, verifica as permissões do usuário e grava os registros no **PostgreSQL**.
3. Após a confirmação da escrita, o Backend dispara um evento/requisição ao **Motor de Inteligência (Python)**.
4. O serviço Python consulta o estado atualizado no PostgreSQL via **Pandas**, recalcula riscos, gargalos e impactos em cascata, e grava os novos alertas/recomendações no banco.
5. O Frontend atualiza os indicadores do Dashboard e notifica a **Central de Alertas**.

### 4.2. Padrão de Comunicação entre Node.js e Python
Para manter o acoplamento baixo e facilitar a implantação em contêineres:
* **Comunicação Síncrona (Sob Demanda):** O Node.js aciona um endpoint REST interno no serviço Python (ex.: `POST /api/intelligence/analyze`) após operações críticas (importação em lote ou atualização de estoque/produção).
* **Processamento Periódico (Batch/Cron):** Rotina agendada que reavalia prazos de entrega diariamente, identificando pedidos que entraram em janela de risco mesmo sem novas movimentações manuais.

---

## 5. Segurança e Controle de Acesso (RBAC)

A segurança da aplicação é estruturada em três pilares:
1. **Autenticação:** Login com credenciais criptografadas (hash `bcrypt`) e emissão de **JSON Web Tokens (JWT)** com tempo de expiração definido.
2. **Autorização (RBAC):** Middleware no Express que valida o perfil do usuário antes de executar cada rota:
   * `Administrador`: Acesso total + gestão de usuários e parâmetros.
   * `Gestor Industrial`: Leitura global de dashboards, relatórios e alertas.
   * `Planejador de Produção`: Leitura/escrita em linhas e ordens de produção.
   * `Responsável pelo Estoque`: Leitura/escrita em matérias-primas e movimentações.
   * `Responsável Comercial`: Leitura/escrita em clientes e pedidos.
   * `Responsável pela Logística`: Leitura/escrita em entregas e expedição.
3. **Proteção de Dados:** Uso de variáveis de ambiente (`.env`) para credenciais de banco, sanitização de inputs contra SQL Injection/XSS e comunicação via HTTPS.

---

## 6. Topologia de Ambiente e Implantação (MVP)

Para padronizar o desenvolvimento da equipe e facilitar a demonstração do ambiente simulado, a arquitetura prevê o empacotamento via **Docker** e **Docker Compose**:

* **Container `frontend`:** Servidor web (Nginx ou Vite/Node dev server) servindo a aplicação React.
* **Container `backend`:** Servidor Node.js + Express na porta `3000`.
* **Container `intelligence`:** Serviço Python (FastAPI + Pandas + Scikit-learn) na porta `8000` (rede interna).
* **Container `database`:** Instância PostgreSQL 15+ com volume persistente para os dados da simulação.

---

## 7. Decisões Arquiteturais e Próximos Passos

| Decisão | Escolha Técnica | Justificativa |
| :--- | :--- | :--- |
| **Separação Backend / IA** | Node.js + Python | Aproveita a agilidade do ecossistema JS para APIs web e o poder do ecossistema Python para ciência de dados. |
| **Banco Relacional** | PostgreSQL | Alta confiabilidade, suporte robusto a transações ACID e consultas analíticas complexas com SQL. |
| **Motor de Regras no MVP** | Heurísticas em Python | Permite validar cenários industriais sem depender de grandes volumes históricos iniciais para treino de IA. |

**Próximos passos técnicos recomendados:**
1. Definir o **Diagrama Entidade-Relacionamento (DER)** detalhado no PostgreSQL.
2. Especificar os contratos de API (Swagger/OpenAPI) entre Frontend, Backend e Motor Python.
3. Configurar o `docker-compose.yml` base para o ambiente de desenvolvimento da equipe.




   


# ⚠️ Critérios de Identificação de Riscos — Industrial Control Tower

**Versão:** 1.0  
**Status:** Especificação de Regras de Negócio (Motor de Inteligência)  
**Módulos Relacionados:** Inteligência Operacional (5.5), Central de Alertas (5.6) e Cenários Operacionais (6)  

---

## 1. Objetivo

Este documento define as regras determinísticas, métricas, fórmulas e faixas de criticidade que o **Motor de Inteligência (Python)** utilizará no MVP para identificar riscos operacionais, detectar gargalos produtivos, mapear impactos entre departamentos e acionar alertas automáticos na plataforma.

---

## 2. Matriz Geral de Criticidade

Todos os riscos identificados pelo sistema são classificados em quatro níveis padronizados para orientar a priorização na **Central de Alertas** e no **Dashboard Executivo**:

| Nível de Criticidade | Código | Cor de Referência | Significado Operacional | Prazo de Ação Recomendado |
| :--- | :--- | :--- | :--- | :--- |
| **Crítico** | `CRITICO` | Vermelho | Impacto iminente ou já em curso na entrega ao cliente ou parada de linha. Exige intervenção imediata. | Imediato (até 4 horas) |
| **Alto** | `ALTO` | Laranja | Forte probabilidade de atraso ou ruptura caso nenhuma medida corretiva seja tomada no curto prazo. | Curto prazo (até 24 horas) |
| **Médio** | `MEDIO` | Amarelo | Desvio operacional detectado, mas ainda há margem de manobra (folga de prazo ou estoque de segurança). | Monitoramento ativo (até 48 horas) |
| **Baixo / Informativo** | `BAIXO` | Azul / Verde | Ponto de atenção preventivo sem impacto direto imediato nos prazos comprometidos. | Rotina de planejamento |

---

## 3. Critérios por Categoria de Risco

### 3.1. Risco de Atraso de Pedido (Comercial & Produção)

Avalia a capacidade de concluir a fabricação e expedição de um pedido antes da data acordada com o cliente.

#### Variáveis Utilizadas:
* $T_{\text{restante}}$: Tempo disponível até o prazo de entrega ($D_{\text{entrega}} - D_{\text{atual}}$), em dias.
* $T_{\text{producao}}$: Tempo estimado para produzir a quantidade pendente ($\frac{Q_{\text{pendente}}}{\text{Capacidade da Linha}}$), em dias.
* $T_{\text{logistica}}$: Tempo padrão de expedição e transporte até o cliente, em dias.
* $T_{\text{necessario}} = T_{\text{producao}} + T_{\text{logistica}}$.
* **Índice de Folga ($IF$):** $IF = T_{\text{restante}} - T_{\text{necessario}}$ (em dias).

#### Regras de Classificação:
1. **Sem Ordem de Produção (OP) vinculada:**
   * Se o pedido está confirmado, não possui OP associada e $T_{\text{restante}} \le T_{\text{necessario}} + 2\text{ dias}$ $\rightarrow$ **Crítico**.
   * Se não possui OP associada e $T_{\text{restante}} \le T_{\text{necessario}} + 5\text{ dias}$ $\rightarrow$ **Alto**.
2. **Com Ordem de Produção vinculada:**
   * **$IF < 0$** (tempo restante menor que o tempo necessário para produzir e entregar) $\rightarrow$ **Crítico**.
   * **$0 \le IF \le 1\text{ dia}$** (margem zero ou mínima; qualquer parada gera atraso) $\rightarrow$ **Alto**.
   * **$1 < IF \le 3\text{ dias}$** (folga reduzida) $\rightarrow$ **Médio**.
   * **$IF > 3\text{ dias}$** $\rightarrow$ Sem risco (Normal).
3. **Atraso na Execução da OP:**
   * Se a data atual ultrapassou a `Data Prevista de Início` e a OP continua com status `Não Iniciada` $\rightarrow$ Eleva automaticamente o risco em **1 nível**.

---

### 3.2. Risco de Falta de Matéria-Prima (Estoque & Suprimentos)

Avalia se o saldo físico e projetado de insumos atende às ordens de produção planejadas e em andamento.

#### Variáveis Utilizadas:
* $E_{\text{atual}}$: Quantidade física disponível no estoque.
* $E_{\text{minimo}}$: Estoque mínimo de segurança cadastrado para o material.
* $D_{\text{comprometida}}$: Soma da quantidade necessária do material para todas as OPs `Planejadas` e `Em Andamento`.
* **Saldo Projetado ($SP$):** $SP = E_{\text{atual}} - D_{\text{comprometida}}$.

#### Regras de Classificação:
* **$SP < 0$ e afeta OP com início em até 48 horas:** Ruptura imediata na linha $\rightarrow$ **Crítico**.
* **$SP < 0$ e afeta OP com início superior a 48 horas:** Déficit confirmado, mas há janela para compra/reposição emergencial $\rightarrow$ **Alto**.
* **$0 \le SP < E_{\text{minimo}}$:** O estoque atende às OPs atuais, mas ficará abaixo do nível de segurança $\rightarrow$ **Médio**.
* **$E_{\text{atual}} \le E_{\text{minimo}}$ (mesmo sem OPs imediatas):** Ponto de ressuprimento atingido $\rightarrow$ **Baixo / Informativo**.

---

### 3.3. Risco de Gargalo Produtivo (Planejamento & Produção)

Avalia a sobrecarga das linhas de produção em um determinado horizonte de tempo (diário ou semanal).

#### Variáveis Utilizadas:
* $C_{\text{disponivel}}$: Capacidade produtiva máxima da linha no período analisado (ex.: metros de cabo/dia ou horas/semana).
* $D_{\text{programada}}$: Soma das quantidades planejadas das OPs alocadas naquela linha no mesmo período.
* **Taxa de Ocupação ($TO$):** $TO = \left(\frac{D_{\text{programada}}}{C_{\text{disponivel}}}\right) \times 100\%$.

#### Regras de Classificação:
* **$TO > 110\%$:** Sobrecarga severa; impossibilidade física de cumprir o cronograma sem horas extras ou realocação $\rightarrow$ **Crítico**.
* **$100\% < TO \le 110\%$:** Linha acima da capacidade nominal $\rightarrow$ **Alto**.
* **$85\% \le TO \le 100\%$:** Linha operando no limite (alta sensibilidade a falhas ou setup) $\rightarrow$ **Médio**.
* **$TO < 85\%$:** Capacidade saudável $\rightarrow$ Normal.

---

### 3.4. Risco de Atraso Logístico (Expedição & Entregas)

Avalia pedidos já produzidos (ou em fase final) que enfrentam restrições para cumprir a janela de entrega.

#### Regras de Classificação:
* Pedido com status `Pronto para Expedição`, sem transporte agendado e com $T_{\text{restante}} < T_{\text{logistica}}$ $\rightarrow$ **Crítico**.
* Pedido em trânsito com ocorrência registrada (ex.: bloqueio de rota, falha de veículo) que supera a margem de entrega $\rightarrow$ **Alto**.
* Pedido com produção concluída em atraso, exigindo ajuste na janela de coleta $\rightarrow$ **Médio**.

---

## 4. Regras de Cruzamento e Impacto em Cascata (Inteligência Operacional)

O diferencial do **Industrial Control Tower** é encadear os riscos individuais acima para mapear dependências entre departamentos (Cenário 4 do Documento Base):

```text
[Gatilho: Falta de Matéria-Prima]
        │
        ├──► 1. Identifica OPs que consomem o insumo (Produção)
        │         └──► Marca OPs como "Bloqueadas / Em Risco"
        │
        ├──► 2. Identifica Pedidos vinculados às OPs afetadas (Comercial)
        │         └──► Recalcula o Índice de Folga (IF) considerando o prazo de reposição do material
        │
        └──► 3. Identifica Entregas vinculadas aos Pedidos (Logística)
                  └──► Gera alerta unificado com a lista de clientes e prazos impactados
```

### Regra de Propagação de Severidade:
* Sempre que uma ocorrência primária (ex.: falta de cobre ou polímero) afetar um pedido com risco `CRITICO` ou `ALTO`, o sistema herda a maior criticidade para o alerta consolidado e vincula todos os IDs afetados (`ID_Material`, `ID_OP`, `ID_Pedido`, `ID_Entrega`).

---

## 5. Motor de Recomendações Automáticas (MVP)

Para cada tipo de risco detectado, o sistema associa recomendações padronizadas baseadas em regras de negócio:

| Tipo de Risco | Condição Detectada | Recomendação Gerada pelo Sistema |
| :--- | :--- | :--- |
| **Atraso de Pedido** | Pedido sem OP e prazo próximo | *"Gerar Ordem de Produção prioritária imediatamente para o item [X]."* |
| **Gargalo Produtivo** | Linha com $TO > 100\%$ e outra linha compatível com $TO < 75\%$ | *"Realocar OP #[Y] da Linha [A] para a Linha [B] (capacidade ociosa disponível)."* |
| **Falta de Material** | $SP < 0$ com múltiplas OPs concorrentes | *"Priorizar saldo disponível para o Pedido #[Z] (maior criticidade) e acionar compra emergencial do insumo [W]."* |
| **Atraso Logístico** | Produção concluída com folga zero para transporte | *"Priorizar carregamento na doca e avaliar frete expresso/dedicado para cumprir prazo do cliente."* |

---

## 6. Estrutura de Dados do Alerta Gerado

Quando um critério é acionado, o Motor de Inteligência persiste um registro na tabela `alertas` do PostgreSQL contendo:

* `id_alerta`: Identificador único.
* `tipo_risco`: `ATRASO_PEDIDO` | `FALTA_MATERIAL` | `GARGALO_PRODUCAO` | `ATRASO_LOGISTICA` | `IMPACTO_CASCATA`.
* `criticidade`: `CRITICO` | `ALTO` | `MEDIO` | `BAIXO`.
* `departamentos_afetados`: Lista (ex.: `["Estoque", "Produção", "Comercial"]`).
* `entidades_relacionadas`: JSON com IDs de referência (`pedido_id`, `op_id`, `material_id`, `linha_id`).
* `descricao_tecnica`: Resumo quantitativo do desvio (ex.: *"Déficit de 450 kg de Cobre Eletrolítico para a OP #104"*).
* `recomendacao`: Ação sugerida ao gestor.
* `status`: `ABERTO` | `EM_ANALISE` | `RESOLVIDO`.

