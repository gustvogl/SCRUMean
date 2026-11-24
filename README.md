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
