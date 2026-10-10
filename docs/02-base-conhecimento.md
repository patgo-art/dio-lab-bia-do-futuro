# **Base de Conhecimento**

## **Dados Utilizados**

A base de conhecimento foi estruturada integrando datasets públicos e consolidados da indústria de games, engenharia de software e finanças com tabelas operacionais mockadas no projeto:

| **Arquivo** | **Formato** | **Utilização no Agente** |
| --- | --- | --- |
| `vgchartz_steam_sales.csv` | CSV | **(VGChartz / SteamSpy Data):** Projeções de receita, preço médio por região, volume de vendas comparativo e cálculo de *breakeven* no lançamento. |
| `software_effort_cocomo.csv` | CSV | **(PROMISE / OpenML Datasets):** Mapeamento de histórico de esforço estimado vs. horas reais e taxas de refação (*rework rate*) em projetos de software/games. |
| `project_budget_runway.json` | JSON | **(StudioBurn Internal):** Limite de orçamento por Sprint/Milestone, faturas de *outsourcing*, saldo em caixa e cálculo determinístico de *runway* (meses de vida do estúdio). |
| `jira_backlog_history.csv` | CSV | **(Jira/Trello Open Export):** Histórico de tarefas, *story points*, custos por hora/função (Dev, Artista 3D, Audio) e métricas de refação por tipo de asset. |

---

## **Adaptações nos Dados**

> Você modificou ou expandiu os dados mockados? Descreva aqui.
> 

Os datasets públicos e as tabelas operacionais foram adaptados e unificados para o cenário de um **estúdio indie de jogos**:

1. **Cruzamento de Horas com Custo Monetário ($/h):** O dataset de esforço de software (*PROMISE/Cocomo*) foi expandido para incluir o custo da hora/homem da equipe interna (ex: Artista 3D Senior a `$55/h`, Tech Art a `$50/h`), permitindo traduzir *story points* diretamente em impacto financeiro.
2. **Atributos de Assets de Jogos:** O histórico do Jira foi enriquecido com metadados técnicos de produção de jogos (ex: *polycount*, tamanho de mapas de textura PBR, *grid snapping*, *LODs*), conectando a complexidade técnica do asset com a taxa de refação.
3. **Mapeamento de Cotações & Asset Store:** Foram adicionados valores médios de mercado de pacotes da *Unreal Engine / Unity Asset Store* para viabilizar as sugestões automáticas de "comprar asset pronto vs. desenvolver do zero".
4. **Tratamento de Dados Incompletos:** Caso um papel técnico ou asset não possua histórico no dataset (ex: áudio ou VFX novo), o `data_manager.py` retorna um status de 'dado ausente', acionando o Guardrail da Agente Gaga para solicitar a taxa horária ao Producer antes de responder.

---

## **Estratégia de Integração**

### **Como os dados são carregados?**

> Descreva como seu agente acessa a base de conhecimento.
> 

Os dados ficam armazenados na pasta `data/` e são manipulados por um módulo Python (`data_manager.py`):

- Arquivos de estado e finanças estáticas (`project_budget_runway.json`) são parseados para objetos Python/Pydantic no início da sessão.
- Datasets volumosos (`vgchartz_steam_sales.csv` e `jira_backlog_history.csv`) são indexados localmente via **SQLite/Pandas**, permitindo que o agente execute consultas SQL e análises estatísticas super rápidas. O `data_manager.py` expõe métodos diretos em Python que funcionam como as **ferramentas (tools)** do agente (ex: `get_role_hourly_rate(role)`, `get_asset_benchmark(category)`)

### **Como os dados são usados no prompt?**

> Os dados vão no system prompt? São consultados dinamicamente?
> 

Os dados **NÃO** são colocados na íntegra dentro do *System Prompt* (para evitar alucinações e estouro de contexto).

- O *System Prompt* contém apenas as instruções de comportamento da **Agente Gaga** e as assinaturas das **Tools (Function Calling)**.
- Quando o Producer faz um pedido (ex: *"Fatie as tarefas da Mina Abandonada"* ou *"Qual o nosso breakeven na Steam?"*), o LLM aciona as ferramentas em Python/SQL (ex: `calculate_burn_rate()`, `get_sprint_budget()`, `query_market_benchmarks()`).
- O código Python executa a matemática exata e injeta apenas os **valores consolidados e ancorados (*grounding*)** no contexto para a IA formatar a resposta.

---

## **Exemplo de Contexto Montado**

> Mostre um exemplo de como os dados são formatados para o agente.
> 

```
==================================================
CONTEXTO OPERACIONAL E FINANCEIRO (STUDIOBURN AI)
==================================================

[METRICAS DE CAIXA & RUNWAY - project_budget_runway.json]
- Projeto: Tyrant's Twilight (Milestone: Alpha / Sprint 12)
- Teto de Orçamento da Sprint 12 (Arte 3D): $ 2.500,00
- Caixa Restante para o Milestone: $ 4.000,00
- Runway Atual do Estúdio: 3.5 meses restantes

[BENCHMARKS DE CUSTO E REFAÇÃO - jira_backlog_history.csv]
- Custo Média Hora/Homem (Arte 3D): $ 50,00/h
- Taxa Média de Refação (Últimas 3 semanas): +15% de horas extras por desvio de polycount
- Benchmark de Props Similares (Unreal Marketplace): $ 45,00 (Pacote Mineração)

[BENCHMARKS DE MERCADO / STEAM - vgchartz_steam_sales.csv]
- Preço Alvo do Jogo: $ 19,99
- Taxa Steam + Impostos: ~37% (Receita Líquida por Cópia: $ 12,60)
- Meta de Breakeven Acumulada: 4.762 cópias

==================================================
SOLICITAÇÃO DO PRODUCER:
"Gaga, bom dia! Preciso criar a lista de tarefas para a equipe de arte 3D referente ao pacote da 'Mina Abandonada'..."
==================================================
```
