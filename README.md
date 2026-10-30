# V-Credit

Este projeto foi desenvolvido como parte do desafio de Engenharia de Dados do **Visagio Rocket Lab 2026.2**, utilizando um cenário de atendimento da empresa fictícia **V-Credit**.

O objetivo é construir um pipeline de dados no **Databricks**, seguindo a arquitetura medalhão (**Bronze, Silver e Gold**), para organizar, tratar e transformar dados brutos em informações estruturadas, consistentes e prontas para análise.

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
docs/
  figures/                    gráficos utilizados para as análise
  testes/                     regras de tratamento de duplicatas
  painel/
    Indicadores_operacao.png
    Qualidade_dados.jpeg
    painel_vcredit.lvdash.json      painel em formato .json
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

## Execução
 
O pipeline roda pelo job `pipeline_vcredit`, que lê os notebooks direto da branch `main` deste repositório. As etapas seguem esta ordem:
 
1. Ingestão: `bronze_ingestion` carrega os 8 CSVs.
2. Silver: os notebooks `silver_*` rodam em paralelo. A exceção é `silver_chamados_hora`, que espera `silver_clientes` e `silver_chamados` terminarem, porque valida os registros contra essas duas tabelas.
3. Gold, base: `gold_fato_chamados`, `gold_chamados_custos` e `gold_chamados_nota` rodam depois que todas as silvers terminam.
4. Gold, painel: `gold_qualidade_dados` e `gold_indicadores` rodam por último, depois de `gold_chamados_nota`.
### Agendamento
 
O job roda a cada 5 dias. Como as fontes são fixas, a frequência foi definida só por padrão e pode ser alterada se passarem a existir fontes atualizadas diariamente.
 
### Reexecução
 
Todas as tabelas são gravadas com `overwrite` (ou `CREATE OR REPLACE`), sempre a partir da camada anterior, então o resultado sobre os mesmos dados é idêntico e não há registros duplicados. A única coluna que muda é o `ingest_timestamp`, que guarda o momento da carga. Nas tabelas gold, um `assert` confirma que continua existindo uma linha por chamado.
Dessa forma, rodar o pipeline de novo é seguro. 
 
### Execução manual
 
Use o Run now do job ou execute os notebooks na ordem acima.

---

## Decisões de tratamento
 
Padronização de texto: Canal e motivo ficam em minúsculas, sem acento e sem espaços extras. Sem isso, os joins entre `chamados`, `canais` e `motivos` perdem linhas por diferenças de grafia.
 
Marcar em vez de remover: Inconsistências que não invalidam o chamado (abandono, atendente placeholder, cliente fora da base) recebem uma flag e permanecem na tabela. Assim nenhum chamado real é perdido e o impacto de cada problema pode ser medido.
 
Abandono: É o chamado sem hora de início de atendimento. Todos ocorrem em canais humanos e não têm tempo de atendimento, por isso são tratados como desistência na fila.
 
Tempos: Os valores registrados de espera e de atendimento não passam de cerca de 999 s (aproximadamente 16 min). Na gold, o tempo adotado é o maior entre o registrado e o calculado pelos horários (`espera_seg` e `atend_seg`). Quando o valor registrado passa de 10 s, ele coincide em 100% dos casos com os horários, o que indica que o cálculo pelos horários é a referência mais confiável.
 
Atendente: URA e chatbot não têm atendente humano, mas a base traz um `id_atendente` aleatório nesses canais. As análises por atendente consideram apenas os canais humanos.
 
Dados pessoais: CPF, nome e e-mail não são usados nas tabelas de análise, e a idade aparece apenas como faixa etária.

---

## Qualidade dos dados

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
Tempo de espera corrigido pelos horários | Recalculado na gold | 6609 | 5%
Chamados com nota inválida na pesquisa (vazia, 0 ou > 5) | Nota anulada | 6626 | 5,2%

**Principal impacto:** o tempo médio de atendimento registrado subestima o real. No nível 2 (atendimento especializado), o registro indica **1,7 min**, mas o tempo real calculado pelos horários é **38,7 min**; no nível 1, 6,1 min registrados contra 16,1 min reais.

Os números completos e atualizados estão no painel.

---

## Painel de indicadores
 
O painel foi feito no Databricks (Dashboards) e lê as tabelas da gold. Ele tem duas páginas.
 
Qualidade dos dados
- Linhas por camada: quantos registros cada tabela tinha na bronze, quantos sobraram na silver e quantos foram removidos.
- Inconsistências tratadas: cada problema encontrado, o tratamento aplicado e quantos chamados afeta.
- Tempo de atendimento, registrado e real, por canal.

Indicadores da operação
- Total de chamados e custo total do período.
- Volume e custo por mês e canal.
- Resolução por canal e abandono nos canais humanos.
- Tempo real de atendimento e satisfação mensal.

---
Link da apresentação: https://www.canva.com/design/DAHZafa8muo/yOwt-H6rV-tqH7UURUnebg/edit

Para ver o painel, importe o arquivo `painel/painel_vcredit.lvdash.json`. Como ele consulta a gold, o pipeline precisa ter rodado antes no mesmo workspace. As capturas de cada página estão em `docs/painel/`.

---
