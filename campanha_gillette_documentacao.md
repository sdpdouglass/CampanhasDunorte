# Documentação Técnica — CAMPANHAS DUNORTE DISTRIBUIDORA.QVS

> Complementa `CAMPANHAS.MD` (regras de negócio) e `METRICAS E DIMENSOES.MD`
> (mapeamento de dimensões/métricas) com o que foi levantado a partir da
> leitura do script de carga real. Objetivo: servir de contexto para
> continuar o desenvolvimento sem precisar reler o `.QVS` inteiro (2008
> linhas) a cada conversa.

## 1. O que este script faz

Pipeline de transformação (ETL) que roda mensalmente/trimestralmente e
gera uma série de QVDs intermediários e finais usados pelos dashboards de
campanhas comerciais (P&G / Mega Marcas) da Dunorte Distribuidora. Não
tem UI — é só carga de dados.

Fontes de origem:
- Planilhas Excel em `lib://4_Plan/Metas P&G/` (`CAMPANHAS_AAAA_MM.xlsx`,
  `METAS_AAAA-MMM.xlsx`) — cadastros de metas, prêmios e faixas.
- QVDs já tratados em `lib://Carga_Duno/TESTE/TRANSFORMADOR/MES/COMERCIAL/MEGA MARCAS/`
  (`COMERCIAL_TRATADO_AAAA_MM.QVD`) — fato de vendas do mês.
- QVDs de outras esteiras (sem `/TESTE/`): Escolha Certa e Platinum Point,
  em `lib://Carga_Duno/TRANSFORMADOR/MES/COMERCIAL/MEGA MARCAS/`.
- Cadastros: `CAD_RCA.QVD`, `CAD_CLIENTE.QVD` em
  `lib://Carga_Duno/EXTRATOR/CADASTRO/`.

Destino final dos QVDs transformados:
`lib://Carga_Duno/TESTE/TRANSFORMADOR/MES/COMERCIAL/MEGA MARCAS/CAMPANHAS/`

## 2. Estrutura em 3 camadas (reorganizado em 2026-08-27)

O script tem 3 abas Qlik (`///$tab`), nesta ordem, cada uma rodando
integralmente antes da próxima:

### Aba "Transformação" — extração/transformação, gera QVDs intermediários
1. **Mappings de cadastro** (`MAP_SUP`, `MAP_RCA_NOME`, `MAP_RCA_SUP`,
   `MAP_CLIENTE_PRINC`) — carregados uma vez, usados via `ApplyMap()` em
   quase todo o resto do script.
2. **TRF_BASE_RCA_INDICADORES** — extrai a planilha CAMPANHAS (abas
   `PREM_RCA` + `INDICADORES`) e gera a base "catálogo" de indicadores
   (todo indicador que existe, independente de ter sido batido ou não).
3. **PREM_RANK_GILLETE** — ranking de prêmio por posição/grupo (Gillette
   Trimestral) + mapeamento supervisor→grupo de rank.
4. **Fato Vendas Trimestre Fixo** (Seção + Departamento) — consolida os 3
   meses do trimestre calendário atual (T1-T4, alinhado ao trimestre
   civil).
4.1. **Fato Vendas Trimestre Móvel** (Seção + Departamento + Devolução
   RCA + Faturamento RCA + Positivação Seção/Departamento) — mesma
   dimensões/medidas do transformador de Vendas Mês Atual, mas somadas
   sobre uma **janela móvel de 3 meses** (mês atual + 2 anteriores,
   recalculada todo mês, diferente do Trimestre Fixo). Usa
   `AddMonths(MonthStart(Today()), offset)` para tratar corretamente a
   virada de ano (ex: Jan/2026 → janela Nov/2025-Jan/2026). Não inclui
   Mix Mínimo/Listing (indicadores de campanha mensal, não agregação de
   vendas).

   A extração bruta (`TMP_VENDAS_TRIMESTRE_MOVEL`) carrega o mesmo
   conjunto completo de campos do `TMP_VENDAS` do Mês Atual (`Cod
   Cliente Principal` via `ApplyMap`, `Cod Produto`, todas as
   quantidades em unidade/caixa de pedido e faturado) - ajustado pelo
   usuário em 2026-08-27 para ficar disponível para uso futuro (ex: um
   Mix Mínimo/Listing ou indicador a nível de produto sobre a janela
   móvel), mesmo que as 6 agregações atuais (itens 3-6 do bloco) só
   usem um subconjunto desses campos por enquanto.

   QVDs: `FATO_VENDAS_SECAO_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_VENDAS_DEPARTAMENTO_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_DEVOLUCAO_RCA_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_FATURAMENTO_RCA_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_POSITIVACAO_SECAO_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_POSITIVACAO_DEPARTAMENTO_TRIMESTRE_MOVEL_AAAA_MM.qvd`.
   Adicionado em 2026-08-27 — **ainda não ligado ao `TRF_BASE_RCA`** (ver
   Pontos em aberto).
5. **Fato Vendas Mês Atual** (Seção + Departamento + RCA) — inclui
   Positivação de Clientes, o cálculo completo de **Mix Mínimo** (seção
   7 do script, ver item 4 abaixo) e de **Listing Iniciativas - 100%
   Carteira** (seção 8, ver item 4.1 abaixo).
6. **Escolha Certa** e **Platinum Point** (mês atual) — QVDs de KPI por
   RCA.
7. **Metas P&G** — lê `METAS_AAAA-MMM.xlsx` (abas `RCA_SEC`, `RCA_DEP`,
   `PLATINUM_POINT`, `ESCOLHA_CERTA`) e gera um QVD de meta por aba.
8. **PREM_RCA** — extrai a aba PREM_RCA (Cod RCA, Indicador, Ganho,
   Faixa) → `TRF_PREM_RCA_AAAA_MM.qvd`.
9. **RCA_DEVOL** — extrai as 4 faixas de % devolução/repasse por RCA →
   `TRF_RCA_DEVOL_AAAA_MM.qvd`.
10. **RCA_DEVOL_RANK** — extrai a faixa máxima de devolução para o
    ranking Gillette → `TRF_RCA_DEVOL_RANK_AAAA_MM.qvd`.

> Os itens 8-10 ficavam antes intercalados dentro da camada de cálculo
> (entre `BASE_RCA_INDICADORES_REALIZADO`, o cálculo de Ganho e o
> Ranking Gillette) — movidos para cá em 2026-08-27, ver Changelog.

### Aba "Modelagem" — onde as bases transformadas se ligam e os cálculos acontecem
1. **TRF_BASE_RCA** (montagem vertical) — uma tabela única
   Data+RCA+Indicador com Meta e Realizado juntos, feita via uma
   sequência de `LEFT JOIN`s com chaves textuais compostas
   (`_Meta`, `_MetaDep`, `_MetaPP`, `_MetaEC`, `_MetaMix`, `_MetaListing`...)
   — cada join usa um nome de campo de valor único para evitar o "bug de
   chave dupla" (ver seção 5). Resultado: `BASE_RCA_REAL_SECAO_DEP_MES_AAAA_MM.qvd`.
2. **BASE_RCA_INDICADORES_REALIZADO** — agrega tudo (Base + Escolha
   Certa + Platinum Point) no grão Data+RCA+Indicador, calcula
   `PercAtingimentoPedido`/`PercAtingimentoFaturado`.
3. **Cálculo do Ganho por RCA/Indicador** — lê o QVD do PREM_RCA (aba
   Transformação), cruza faixa de premiação com o % de atingimento
   realizado → `GanhoPedido`/`GanhoFaturado`.
4. **% Devolução RCA + Faixa de Repasse + Ganho Final** — lê os QVDs do
   RCA_DEVOL (aba Transformação), calcula % devolução, faixa de repasse
   aplicável e `GanhoFinalPedidoRca`/`GanhoFinalFaturadoRca`.
5. **Ranking Gillette Trimestral** — lê os QVDs do RCA_DEVOL_RANK e do
   PREM_RANK_GILLETE (aba Transformação); só entram no ranking RCAs com
   ≥100% de atingimento; RCAs com devolução acima da faixa máxima
   permitida são zerados no ranking (mas não no ganho por indicador).
   Resultado final: `PREMIACAO_GILLETTE_TRI`.

### Aba "Carregamento" — só organização/documentação (sem STORE novo)
Marca `BASE_RCA_INDICADORES_REALIZADO` (com Ganho) e
`PREMIACAO_GILLETTE_TRI` como as duas tabelas finais que carregam no
modelo de dados do app Qlik. Decisão do usuário: essa camada não grava
nada em QVD — as duas tabelas já são o resultado final da Modelagem e
ficam residentes, associadas ao restante do modelo.

## 3. Indicadores implementados no script (x regras de negócio)

| Indicador (CAMPANHAS.MD)              | Implementado no `.QVS`?                                  |
|----------------------------------------|-----------------------------------------------------------|
| Campanha Gillette (Trimestral)         | Sim — seções "Fato Vendas Trimestre Fixo" + Ranking Gillette |
| Campanha Mensal (indicadores gerais)   | Sim — Fato Vendas Mês Atual (Seção/Departamento/RCA)       |
| Regra de devolução (Gillette e Mensal) | Sim — blocos RCA_DEVOL e RCA_DEVOL_RANK                    |
| Mix Mínimo Contrato                    | Sim — seção 7 completa (ver item 4)                        |
| Escolha Certa Especial (R$20/positivação, só RCA) | **Parcial** — o script gera o KPI `FAIXA ESCOLHA CERTA` (soma de `QTD_ESCOLHA_CERTA` por faixa), mas o cálculo do valor R$20 por positivação não aparece neste `.QVS`. Verificar se é feito em outra camada (dashboard/expressão) ou está faltando. |
| Indicador Listing (`LISTING INICIATIVAS--100% CARTEIRA`) | **Sim (implementado e ligado ao TRF_BASE_RCA em 2026-08-26)** — seção 8 do script (cálculo) + seções 3.2/8.2 (Realizado/Meta ligados à base unificada), ver item 4.1 abaixo. Regra: cliente compra TODOS os produtos da aba `LISTING_PRODUTOS` do seu `RAMO`, cada produto exige qtd mínima em CAIXAS; RCA só ganha se 100% da carteira completar. Validado rodando no Qlik Sense. |
| Campanha PET (Supervisor Suzy 240+340→240340) | **Não encontrado neste arquivo.** Nem o indicador PET nem o código fictício 240340 aparecem no script lido. Pode estar em outro `.qvs`/tab do projeto Qlik. |
| Supervisor Gerson (73+74→7374)         | **Não encontrado neste arquivo.** Mesma observação acima. |

> Ação sugerida: confirmar com quem mantém o app Qlik se PET e os
> supervisores fictícios (7374/240340) vivem em outro script/tab antes de
> assumir que estão faltando.

## 4. Mix Mínimo — como funciona no script (seção 7, dentro do bloco "Mês Atual")

Implementa a regra do `CAMPANHAS.MD`: cliente principal precisa positivar
um número mínimo de **grupos de produto** dentro da sua **Categoria**
(DPP/CC/HFS/NMR), e o RCA só ganha se **todos** os seus clientes
principais baterem (regra tudo-ou-nada).

Passo a passo (nomes de tabela no script):
1. `TRF_MIX_MIN` — Cliente Principal × Categoria × Objetivo (nº de grupos
   a bater), vindo da aba `MIX_MIN`.
2. `TRF_MIXMIN_PRODUTO_GRUPO` / `TRF_MIXMIN_GRUPO` — mapeamento
   Produto→Grupo e Grupo→QtdMin, vindo da aba `MIXMIN_GRUPO_PRODUTO`
   (colunas mescladas na planilha original, tratadas com cuidado).
3. `MIXMIN_ELEGIVEL` — join Cliente×Grupo por Categoria (todo cliente
   pareado com todos os grupos da própria categoria).
4. `TEMP_VENDAS_MIXMIN` → `COMPRA_CLIENTE_GRUPO` — quantidade realmente
   comprada por Cliente Principal + Grupo (só conta valor > 0).
5. `MIXMIN_GRUPO_FLAG` — grupo positivado se `QtdComprada >= QtdMin`.
6. `MIXMIN_CLIENTE_FLAG` — cliente atingiu objetivo se
   `GruposPositivados >= Objetivo`.
7. `FATO_MIXMINIMO_RCA` — `Min()` do flag por cliente agregado por RCA:
   só fica 1 (100%) se **todos** os clientes do RCA bateram.

QVD final: `FATO_MIXMINIMO_RCA_MES_AAAA_MM.qvd`. Esse mesmo QVD alimenta
tanto o Realizado quanto a Meta do indicador "MIX MINIMO" na montagem de
`TRF_BASE_RCA` (é ligado duas vezes, com chaves `_RealizadoMix`/`_MetaMix`
separadas).

## 4.1 Listing Iniciativas - 100% Carteira — como funciona no script (seção 8, logo após o Mix Mínimo)

Regra: cliente principal precisa comprar **todos** os produtos da aba
`LISTING_PRODUTOS` que pertencem ao seu **Ramo** (Ramo do produto vs.
Ramo do cliente, este último vindo da aba `MIX_MIN`), cada produto
exigindo uma quantidade mínima em **caixas** (`QTDMIN_CX`). RCA só ganha
se **100% da carteira** (todos os seus clientes principais) completar —
regra tudo-ou-nada, igual ao Mix Mínimo.

Headers reais confirmados na planilha `CAMPANHAS_AAAA_MM.xlsx`:
- `MIX_MIN`: `DATA, COD_SUP, COD_RCA, CODCLIPRINC, CLASSE, RAMO, CATEGORIA, OBJETIVO`
- `LISTING_PRODUTOS`: `DATA, COD_PRODUTO, NOMEPRODUTO, RAMO, QTDMIN_CX`

Passo a passo (nomes de tabela no script, seção 8.1 a 8.7):
1. `LISTING_PARTICIPANTES` — Cliente Principal × RCA × Ramo, mesma base
   de participantes da aba `MIX_MIN` (reload minimalista — só os 3
   campos necessários, já que `TRF_MIX_MIN` da seção 7 não preserva o
   campo `Ramo` até aqui).
2. `TRF_LISTING_PRODUTOS` — Produto × Ramo × QtdMinCx, vindo da aba
   `LISTING_PRODUTOS`.
3. `LISTING_ELEGIVEL` — join Cliente×Produto por Ramo (todo cliente
   pareado com todos os produtos obrigatórios do próprio ramo).
4. `TEMP_VENDAS_LISTING` → `COMPRA_CLIENTE_PRODUTO_LISTING` — quantidade
   em **caixas** realmente comprada por Cliente Principal + Produto (só
   conta valor > 0). Usa `QtdCaixaPedidoLiquido`/`QtdCaixaFaturadoLiquido`
   de `TMP_VENDAS` (não as quantidades em unidades, usadas pelo Mix
   Mínimo) — decisão confirmada com o usuário, já que `QTDMIN_CX` é
   literalmente "quantidade mínima de caixas".
5. `LISTING_PRODUTO_FLAG` — produto positivado se
   `QtdCaixaComprada >= QtdMinCx`.
6. `LISTING_CLIENTE` — cliente completou a lista se `Min()` de todos os
   flags de produto do seu Ramo = 1 (um produto faltando já derruba).
7. `FATO_LISTING_RCA` — `Min()` do flag por cliente agregado por RCA:
   só fica 1 (100%) se **toda a carteira** do RCA completou.

QVD final: `FATO_LISTING_RCA_MES_AAAA_MM.qvd`, com
`Indicador='LISTING INICIATIVAS--100% CARTEIRA'`,
`TipoIndicador='DEPARTAMENTO'`, `ClasseIndicador='MANUAL'`, `Meta=1`
fixo — mesmo padrão de campos do Mix Mínimo.

> **Atenção ao nome exato do indicador**: a linha de catálogo na aba
> `INDICADORES` da planilha `CAMPANHAS_AAAA_MM.xlsx` usa literalmente
> `LISTING INICIATIVAS--100% CARTEIRA` (dois hífens, sem espaço, "100%"
> colado em "CARTEIRA") — **não** `LISTING INICIATIVAS`. Como a chave
> composta (`_RealizadoListing`/`_MetaListing`) exige match textual
> exato do campo `Indicador` contra essa linha do catálogo, usar o nome
> "limpo" quebraria o join silenciosamente (Meta/Realizado ficariam
> `Null` sem erro nenhum). Confirmado lendo `CAMPANHAS_2026_08.xlsx`
> diretamente (aba `INDICADORES`: `CODIGO=3, TIPO=DEPARTAMENTO,
> CLASSE=MANUAL, PERIODO=MESATUAL`).

**Ligado ao `TRF_BASE_RCA`** (2026-08-26, seções 3.2 e 8.2 do script,
mesmo padrão do Mix Mínimo):
- `REALIZADO_LISTING` — lê `FATO_LISTING_RCA_MES_$(vDataCarg).qvd`,
  chave `_RealizadoListing`, valores renomeados
  `ValorPedidoLiquidoListing`/`ValorFaturadoLiquidoListing`.
- `META_LISTING` — mesma fonte, chave `_MetaListing`, valor
  `MetaListing` (= `Meta`, sempre 1).
- Etapa 9 (UNIFICAÇÃO) atualizada: `MetaListing` entrou no `Alt()` de
  `Meta`, `ValorPedidoLiquidoListing`/`ValorFaturadoLiquidoListing`
  entraram nos `Alt()` de `ValorPedidoLiquidoFinal`/
  `ValorFaturadoLiquidoFinal`, e todos os campos intermediários foram
  incluídos no `DROP FIELD` final.

Resultado: "LISTING INICIATIVAS--100% CARTEIRA" agora aparece
normalmente em `BASE_RCA_REAL_SECAO_DEP_MES_AAAA_MM.qvd`, com Meta e
Realizado, junto dos demais indicadores.

## 5. Convenções importantes do script (para não quebrar nada em manutenção)

- **Chaves compostas com nome de campo único por join**: toda vez que um
  novo bloco de Meta é `LEFT JOIN`ado em `TRF_BASE_RCA`, o campo de valor
  vem renomeado (`MetaSecaoFat`, `MetaDepFat`, `MetaPlatinumPoint`...).
  Isso evita o "bug de chave dupla" do Qlik: se dois LEFT JOINs
  sucessivos compartilhassem o mesmo nome de campo de valor, esse campo
  viraria parte da chave de match do segundo join sem querer. Ao criar
  um novo bloco de Meta/Realizado, sempre dar um nome de campo de valor
  exclusivo antes do join.
- **Indicadores que dividem `TipoIndicador`/`Codigo`**: Platinum Point,
  Escolha Certa e Mix Mínimo compartilham `TipoIndicador='DEPARTAMENTO'`
  e `Codigo='3'` na base de indicadores — por isso o nome do `Indicador`
  entra na fórmula da chave (`_MetaPP`, `_MetaEC`, `_MetaMix`) para não
  colidir.
- **Caminhos**: `vPathMetasPG`, `vPathCampanhas`, `vPathVendasMes` foram
  centralizados no topo do script (ver Changelog). Escolha Certa e
  Platinum Point são exceção — leem de fora de `/TESTE/`
  (`lib://Carga_Duno/TRANSFORMADOR/...`), propositalmente diferente do
  resto.
- **Padrão de bloco opcional**: quase todo bloco de extração é envolvido
  em `IF Not IsNull(FileSize(...)) THEN ... ELSE TRACE aviso ENDIF` —
  se o arquivo de origem do mês/trimestre ainda não existe, o bloco é
  pulado sem quebrar o restante da carga (útil pra rodar o script antes
  do fechamento do mês).
- **Vigência sempre = mês/trimestre atual**: todas as datas são derivadas
  de `Today()`. Não há parâmetro para reprocessar um mês passado sem
  editar o script manualmente — se isso virar necessidade recorrente,
  vale adicionar uma variável de override no topo.

## 6. Changelog de otimizações aplicadas (2026-08-26)

1. **Eliminado round-trip de disco no bloco Mix Mínimo.** `TRF_MIX_MIN`,
   `TRF_MIXMIN_PRODUTO_GRUPO` e `TRF_MIXMIN_GRUPO` deixaram de ser
   gravadas em QVD e imediatamente relidas do disco — agora permanecem
   residentes em memória e são reaproveitadas direto pelas seções
   seguintes. Os QVDs entregáveis continuam sendo gerados normalmente.
2. **Caminhos de origem/destino centralizados** em três constantes no
   topo do script (`vPathMetasPG`, `vPathCampanhas`, `vPathVendasMes`),
   substituindo ~10 declarações `SET` idênticas espalhadas pelo arquivo.
   Os blocos de Escolha Certa/Platinum Point mantiveram seu path próprio
   (origem diferente, sem `/TESTE/`).
3. **Corrigido comentário desatualizado** na seção "BASE DE INDICADORES
   (bruto)" que descrevia uma conversão texto→número (vírgula decimal)
   que não existe em nenhum ponto do código — os valores já chegam
   numéricos (produzidos por `Sum()` mais acima no próprio script).
   Substituído por nota + `// TODO: verify` para o caso da fonte mudar.

Nenhuma lógica de negócio, nome de campo ou QVD de saída foi alterado —
mudanças puramente estruturais/de I/O e de documentação. **Ainda não
validado rodando no Qlik Cloud** — validar antes de subir para produção.

**2026-08-26 (2ª leva):** Implementado o cálculo do Realizado da campanha
**Listing Iniciativas--100% Carteira** (seção 8, novo bloco — ver item
4.1). QVD gerado: `FATO_LISTING_RCA_MES_AAAA_MM.qvd`. Validado rodando
no Qlik Sense (script completo, sem erros).

**2026-08-26 (3ª leva):** Ligado o `FATO_LISTING_RCA` ao encadeamento de
`LEFT JOIN`s de `TRF_BASE_RCA` (novas seções 3.2 `REALIZADO_LISTING` e
8.2 `META_LISTING`, mesmo padrão do Mix Mínimo) — Listing Iniciativas
agora aparece na tabela unificada `BASE_RCA_REAL_SECAO_DEP_MES_AAAA_MM.qvd`
junto com Meta e Realizado. Corrigido também um mismatch de nome:
o valor correto do campo `Indicador` é `LISTING INICIATIVAS--100%
CARTEIRA` (conforme a linha de catálogo real na aba `INDICADORES`), não
`LISTING INICIATIVAS` — usar o nome errado quebraria o join
silenciosamente. **Validado rodando no Qlik Sense em 2026-08-27
(junto com a reorganização abaixo) — valores batendo.**

**2026-08-27:** Reorganizado o script inteiro em 3 abas Qlik (`///$tab`)
— **Transformação**, **Modelagem**, **Carregamento** — ver seção 2.
Os blocos de extração PREM_RCA, RCA_DEVOL e RCA_DEVOL_RANK, que antes
ficavam intercalados dentro da camada de cálculo, foram movidos
(cut+paste puro, sem alterar nenhuma linha de lógica) para o final da
aba Transformação. Mudança é 100% estrutural — confirmado que os 3
blocos são autocontidos (variáveis próprias baseadas em `Today()`, sem
depender de estado deixado por outros blocos) e que seus consumidores
na Modelagem continuam funcionando porque variáveis Qlik são globais e
persistem independente de onde o bloco está fisicamente no arquivo,
desde que Transformação rode inteira antes de Modelagem. Aba
Carregamento é só documentação (nenhum STORE novo, por decisão do
usuário). **Validado rodando no Qlik Sense — script completo, sem
erros, valores batendo com a versão anterior.**

**2026-08-27 (2ª leva):** Criado o transformador **Vendas Trimestre
Móvel** (aba Transformação, logo após o Trimestre Fixo) — mesmas 6
saídas do transformador de Vendas Mês Atual (Seção, Departamento,
Devolução RCA, Faturamento RCA, Positivação Seção, Positivação
Departamento), somadas sobre a janela móvel de 3 meses (mês atual + 2
anteriores). Ver item 4.1 da seção 2. **Ainda não validado no Qlik e
ainda não ligado ao `TRF_BASE_RCA`** — ver Pontos em aberto.

## 7. Pontos em aberto para continuar o projeto

- Confirmar onde vivem os indicadores **PET** e os supervisores
  fictícios **Gerson (7374)** / **Suzy (240340)** — não estão neste
  `.QVS`.
- Confirmar se o valor de R$20 por positivação do **Escolha Certa
  Especial** é calculado em algum lugar (script ou app Qlik) — não
  localizado aqui.
- Validar no Qlik Sense/Cloud o novo transformador **Vendas Trimestre
  Móvel** (reload completo, conferir os 6 QVDs gerados).
- **Vendas Trimestre Móvel ainda não está ligado ao `TRF_BASE_RCA`** —
  hoje só existe como transformador (aba Transformação). Para aparecer
  na tabela unificada do dashboard, precisaria: (1) uma linha de
  catálogo com `PERIODO='TRIMESTRE MOVEL'` na aba `INDICADORES` da
  planilha CAMPANHAS (mesmo cuidado que tivemos com o nome exato do
  Listing Iniciativas), e (2) mais um `CONCATENATE` na montagem de
  `REALIZADO_SECAO` (Modelagem), igual ao que já existe para Trimestre
  Fixo e Mês Atual.
- O script assume `Today()` como referência de mês/trimestre em ~10
  pontos diferentes — se for necessário reprocessar meses fechados,
  vale adicionar parametrização.
