# V-Credit

Pipeline de dados da operação de atendimento da V-Credit (case Visagio Rocket Lab 2026), com **arquitetura medalhão** (bronze → silver → gold).

O objetivo é transformar as bases brutas de atendimento (jan/2025 a jul/2026) em dados **confiáveis, rastreáveis e atualizados automaticamente**, que servem de base para o painel de indicadores e para as análises de negócio.

---

## Arquitetura

```
CSVs de origem (8 bases)
        │
   ┌────▼─────┐   dados como vieram da fonte (tudo string) + colunas de rastreabilidade
   │  BRONZE  │   rocket_lab.vcredit_bronze
   └────┬─────┘
   ┌────▼─────┐   limpeza, padronização, tipagem e validação; inconsistências removidas ou marcadas
   │  SILVER  │   rocket_lab.vcredit_silver
   └────┬─────┘
   ┌────▼─────┐   tabelas integradas e métricas prontas para análise e painel
   │   GOLD   │   rocket_lab.vcredit_gold
   └────┬─────┘
        ▼
  Painel de indicadores (Databricks Dashboards)
```

---

## Estrutura do repositório

```
README.md
config/
  paths.py                    caminhos dos CSVs de origem
notebooks/
  1_bronze/
    bronze_ingestion          ingestão dos 8 CSVs
  2_silver/
    silver_atendentes
    silver_canais
    silver_chamados
    silver_chamados_hora
    silver_clientes
    silver_custos
    silver_motivos
    silver_pesquisa
  3_gold/
    gold_fato_chamados        tabela consolidada de chamados
    gold_chamados_custos      chamados + motivos + custos + canais
    gold_chamados_nota        base de satisfação (nota, motivo e tempos reais)
    gold_qualidade_dados      métricas de qualidade dos dados (painel)
    gold_indicadores          indicadores mensais da operação (painel)
dashboards/
  painel_vcredit.lvdash.json  export do painel de indicadores
docs/
  diagnostico_silver          verificações usadas para identificar as inconsistências
  testes/                     regras de tratamento de duplicatas
```

Cada notebook começa com uma célula de documentação no mesmo padrão: origem, destino, grão, dependências, regras de tratamento e decisões.

---

## Tabelas

| Camada | Tabela | Grão |
|---|---|---|
| Bronze | `atendentes`, `canais`, `chamados`, `chamados_hora`, `clientes`, `custos`, `motivos`, `pesquisa` | Uma linha por registro da origem |
| Silver | as mesmas 8 tabelas, limpas | Uma linha por registro válido |
| Gold | `fato_chamados` | Uma linha por chamado |
| Gold | `chamados_custos` | Uma linha por chamado |
| Gold | `chamados_nota` | Uma linha por chamado |
| Gold | `qualidade_tabelas`, `qualidade_indicadores` | Uma linha por tabela / por inconsistência |
| Gold | `indicadores_mensais` | Uma linha por mês e canal |

---

## Execução (job)

O pipeline roda pelo job **`pipeline_vcredit`**, que lê os notebooks direto da branch `main` deste repositório e respeita as dependências entre eles:

```
bronze_ingestion
   │
   ├─ silver_canais, silver_motivos, silver_atendentes,
   │  silver_clientes, silver_custos, silver_pesquisa, silver_chamados   (em paralelo)
   │
   ├─ silver_chamados_hora            (depende de silver_clientes e silver_chamados)
   │
   ├─ gold_fato_chamados, gold_chamados_custos, gold_chamados_nota      (dependem de todas as silvers)
   │
   └─ gold_qualidade_dados, gold_indicadores                            (dependem de gold_chamados_nota)
```

- **Agendamento:** a cada 5 dias. Como as fontes são cargas fixas, a frequência pode ser ajustada quando houver fontes atualizadas diariamente.
- **Idempotência:** todas as tabelas são gravadas com `overwrite` (ou `CREATE OR REPLACE`), lendo sempre da camada anterior. Reexecutar o pipeline sobre os mesmos dados produz o mesmo resultado, sem duplicar registros. A única coluna que muda a cada execução é `ingest_timestamp`, que registra o momento da carga.
- **Integridade:** as tabelas gold verificam (`assert`) que mantêm exatamente uma linha por chamado.

Para rodar manualmente, execute os notebooks na ordem acima, ou use **Run now** no job.

---

## Principais regras e decisões

- **Padronização de texto:** canal e motivo ficam sem acento, em minúsculo e sem espaços sobrando nas tabelas de chamados, canais e motivos, para que os joins funcionem.
- **Marcar em vez de apagar:** inconsistências que não invalidam o chamado são marcadas com colunas de flag (abandono, atendente placeholder, cliente fora da base), e não removidas, para não perder chamados reais e permitir medir o impacto.
- **Abandono:** chamado sem hora de início de atendimento (todos em canais humanos e sem tempo de atendimento) é tratado como desistência na fila.
- **Tempos reais:** o tempo de espera e o de atendimento registrados não passam de ~999 s (~16 min). Na gold, os tempos são o maior valor entre o registrado e o calculado pelos horários (`espera_seg`, `atend_seg`). Onde o registro passa de 10 s, ele bate 100% com os horários.
- **Atendente:** URA e chatbot não têm atendente humano, mas os dados trazem um `id_atendente` aleatório nesses canais; análises por atendente consideram só os canais humanos.
- **Dados pessoais:** CPF, nome e e-mail não são usados nas tabelas de análise; a idade é usada como faixa etária.

---

## Qualidade dos dados (resultado dos tratamentos)

| Inconsistência | Tratamento | Chamados | % |
|---|---|---|---|
| Chamados duplicados na origem | Removidos (cópias idênticas) | 2.636 | 2,0% |
| Chamados com horário impossível | Removidos da `chamados_hora` | 199 | 0,15% |
| Abandonos (sem hora de início) | Mantidos e marcados | 3.408 | 2,6% |
| Chamados sem custo | Mantidos; fora das contas de custo | 1.550 | 1,2% |
| Cliente fora da base limpa de clientes | Mantidos e marcados | 2.326 | 1,8% |
| Cliente diferente entre `chamados` e `chamados_hora` | Mantidos | 1.327 | 1,0% |
| Atendente placeholder (0, 99, 999) | Mantidos e marcados | 764 | 0,6% |
| Tempo de atendimento corrigido pelos horários | Recalculado na gold | 28.989 | 22,0% |

**Principal impacto:** o tempo médio de atendimento registrado subestima o real. No nível 2 (atendimento especializado), o registro indica **1,7 min**, mas o tempo real calculado pelos horários é **38,7 min**; no nível 1, 6,1 min registrados contra 16,1 min reais.

Os números completos e atualizados estão no painel.

---

## Painel de indicadores

**Link:** [inserir link do painel publicado]

- **Qualidade dos dados:** linhas por camada, inconsistências encontradas e tratamento, e tempo registrado × real.
- **Indicadores da operação:** volume, custo, resolução, abandono, tempo real de atendimento e satisfação, por mês e por canal.

O painel lê as tabelas da gold e reflete os dados da última execução do job. O arquivo `dashboards/painel_vcredit.lvdash.json` permite recriá-lo em outro workspace (Dashboards > Import).

---

## Equipe

- [nome] · bronze
- [nome] · silver e gold
- [nome] · silver e gold
- [nome] · regras de tratamento de duplicatas