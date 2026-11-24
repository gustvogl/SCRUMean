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