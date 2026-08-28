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
3. **PREM_RANK_GILLETTE_RCA** — ranking de prêmio por posição/grupo
   (Gillette Trimestral, nível RCA) + mapeamento supervisor→grupo de
   rank. Renomeado de `PREM_RANK_GILLETE` (sem `_RCA`) em 2026-08-27
   quando a aba da planilha foi renomeada.
3.1. **PREM_RANK_GILLETTE_SUP** — mesma estrutura do item 3, mas ranking
   a nível de Supervisor (aba nova na planilha). Adicionado em
   2026-08-27, **só extração/transformação por enquanto** — ainda não
   ligado a nenhum cálculo de premiação (uso futuro, conforme pedido do
   usuário). QVDs: `TRF_PREM_RANK_GILLETTE_SUP_AAAA_MM.qvd`,
   `TRF_SUP_GRUPO_GILLETTE_SUP_AAAA_MM.qvd`.
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
| Mix Mínimo Contrato                    | Sim — seção 7, dentro do bloco Vendas Trimestre Móvel desde 2026-08-27 (ver item 4) |
| Escolha Certa Especial (R$20/positivação, só RCA) | **Parcial** — o script gera o KPI `FAIXA ESCOLHA CERTA` (soma de `QTD_ESCOLHA_CERTA` por faixa), mas o cálculo do valor R$20 por positivação não aparece neste `.QVS`. Verificar se é feito em outra camada (dashboard/expressão) ou está faltando. |
| Indicador Listing (`LISTING INICIATIVAS--100% CARTEIRA`) | **Sim (implementado e ligado ao TRF_BASE_RCA em 2026-08-26)** — seção 8 do script (cálculo) + seções 3.2/8.2 (Realizado/Meta ligados à base unificada), ver item 4.1 abaixo. Regra: cliente compra TODOS os produtos da aba `LISTING_PRODUTOS` do seu `RAMO`, cada produto exige qtd mínima em CAIXAS; RCA só ganha se 100% da carteira completar. Validado rodando no Qlik Sense. |
| Campanha PET (Supervisor Suzy 240+340→240340) | **Não encontrado neste arquivo.** Nem o indicador PET nem o código fictício 240340 aparecem no script lido. Pode estar em outro `.qvs`/tab do projeto Qlik. |
| Supervisor Gerson (73+74→7374)         | **Não encontrado neste arquivo.** Mesma observação acima. |

> Ação sugerida: confirmar com quem mantém o app Qlik se PET e os
> supervisores fictícios (7374/240340) vivem em outro script/tab antes de
> assumir que estão faltando.

## 4. Mix Mínimo — como funciona no script (seção 7, dentro do bloco "Vendas Trimestre Móvel")

Implementa a regra do `CAMPANHAS.MD`: cliente principal precisa positivar
um número mínimo de **grupos de produto** dentro da sua **Categoria**
(DPP/CC/HFS/NMR), e o RCA só ganha se **todos** os seus clientes
principais baterem (regra tudo-ou-nada).

> **Mudança de regra de negócio em 2026-08-27** (a pedido do usuário,
> depois de validar a nova estrutura da planilha):
> 1. **Realizado agora é medido sobre a janela móvel de 3 meses**, não
>    mais só o mês atual — o cálculo inteiro foi realocado do bloco
>    "Vendas Mês Atual" para o bloco "Vendas Trimestre Móvel", e passou
>    a consumir `TMP_VENDAS_TRIMESTRE_MOVEL` em vez de `TMP_VENDAS`.
> 2. **`QTDMIN` agora vem da aba `MIX_MIN`** (um valor por **cliente**,
>    aplicado igual a todos os grupos que ele precisa comprar) — antes
>    vinha da aba `MIXMIN_GRUPO_PRODUTO` (um valor por **grupo**, igual
>    pra todo cliente da mesma categoria). Confirmado direto na
>    planilha: `MIXMIN_GRUPO_PRODUTO` não tem mais coluna `QTDMIN`, e
>    `NOMEGRUPO`/`CATEGORIA` agora vêm preenchidos em toda linha do
>    grupo (antes só na 1ª linha, efeito de célula mesclada) — por isso
>    a extração desses dois campos trocou de
>    `WHERE Not IsNull(QTDMIN)` para `LOAD DISTINCT`.
> 3. **Chave composta Cliente+Filial** (`_ChaveClienteFilial = CodClientePrincipal & '|' & CodFilial`)
>    em toda comparação de compra — confirmado direto na planilha que o
>    mesmo `CODCLIPRINC` pode representar clientes diferentes em filiais
>    diferentes (ex: cliente `334788` aparece nas filiais `1` e `3` na
>    aba `MIX_MIN`). Usar só `CodClientePrincipal` misturaria as compras
>    desses dois clientes.
>
> **O contrato de saída não mudou**: mesmo nome de QVD
> (`FATO_MIXMINIMO_RCA_MES_AAAA_MM.qvd`), mesmos campos
> (`Indicador='MIX MINIMO'`, `TipoIndicador='DEPARTAMENTO'`,
> `PeriodoIndicador='MESATUAL'`, `ClasseIndicador='MANUAL'`, `Meta=1`).
> A Modelagem (`REALIZADO_MIXMINIMO`/`META_MIXMINIMO`) não precisou de
> nenhuma alteração.

Passo a passo (nomes de tabela no script):
1. `TRF_MIX_MIN` — Cliente Principal × Filial × Categoria × Objetivo ×
   **QtdMin** (agora por cliente), vindo da aba `MIX_MIN`, mais o campo
   `_ChaveClienteFilial`.
2. `TRF_MIXMIN_PRODUTO_GRUPO` / `TRF_MIXMIN_GRUPO` — mapeamento
   Produto→**NomeGrupo** e NomeGrupo→Categoria (sem QtdMin nem
   CodGrupo — a chave de negócio do grupo é o nome, não o código
   numérico, que não é único entre categorias), vindo da aba
   `MIXMIN_GRUPO_PRODUTO`.
3. `MIXMIN_ELEGIVEL` — join Cliente(+Filial)×**NomeGrupo** por Categoria
   (todo cliente pareado com todos os grupos da própria categoria);
   `QtdMin` já vem do próprio cliente, sem precisar de join extra.
4. `TEMP_VENDAS_MIXMIN` → `COMPRA_CLIENTE_GRUPO` — quantidade realmente
   comprada por **Cliente+Filial** + **NomeGrupo**, **na janela móvel de
   3 meses** (só conta valor > 0).
5. `MIXMIN_GRUPO_FLAG` — grupo positivado se
   `QtdComprada >= QtdMin (do cliente)`.
6. `MIXMIN_CLIENTE_FLAG` — cliente (+filial) atingiu objetivo se
   `GruposPositivados >= Objetivo`.
7. `FATO_MIXMINIMO_RCA` — `Min()` do flag por cliente+filial agregado
   por RCA: só fica 1 (100%) se **todos** os clientes (em qualquer
   filial) do RCA bateram.

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
anteriores). Ver item 4.1 da seção 2. A extração bruta foi depois
ajustada pelo usuário para carregar o conjunto completo de campos do
Mês Atual (`Cod Cliente Principal`, `Cod Produto`, quantidades
unidade/caixa) para uso futuro. **Validado rodando no Qlik Sense —
script editor carregou normalmente, sem erros, com as 3 abas
(Transformação/Modelagem/Carregamento) reconhecidas corretamente.**
Ainda não ligado ao `TRF_BASE_RCA` — ver Pontos em aberto.

**2026-08-27 (3ª leva):** A planilha `CAMPANHAS_AAAA_MM.xlsx` mudou de
estrutura: a aba `PREM_RANK_GILLETE` foi renomeada para
`PREM_RANK_GILLETTE_RCA`, e uma aba nova `PREM_RANK_GILLETTE_SUP` foi
adicionada (mesmo layout, ranking a nível de Supervisor em vez de RCA).
Script atualizado: `table is PREM_RANK_GILLETE` → `PREM_RANK_GILLETTE_RCA`
nos dois LOADs existentes (senão o reload quebraria por não achar a
aba antiga), e criado um novo bloco espelhado para
`PREM_RANK_GILLETTE_SUP` (`TRF_PREM_RANK_GILLETTE_SUP`/
`TRF_SUP_GRUPO_GILLETTE_SUP`) — só extração por enquanto, sem ligação
com cálculo de premiação (uso futuro, a pedido do usuário). Ver item
3.1 da seção 2.

> `COD_FILIAL`/`QTDMIN` novos na aba `MIX_MIN` (mencionados acima):
> **uso confirmado e implementado em 2026-08-27 (4ª leva)**, ver abaixo.

**2026-08-27 (4ª leva):** Mudança de regra de negócio no indicador
**Mix Mínimo**, a pedido do usuário — ver detalhes completos na seção 4:
(1) o cálculo inteiro foi **movido do bloco "Vendas Mês Atual" para
"Vendas Trimestre Móvel"** (Realizado agora soma 3 meses, não 1);
(2) `QTDMIN` passou a vir da aba `MIX_MIN` (por cliente) em vez de
`MIXMIN_GRUPO_PRODUTO` (por grupo) — confirmado direto na planilha que
essa coluna foi removida de `MIXMIN_GRUPO_PRODUTO`; (3) introduzida a
chave composta `_ChaveClienteFilial` em toda comparação de compra, já
que o mesmo `CODCLIPRINC` pode repetir em filiais diferentes
(confirmado: cliente `334788` aparece nas filiais `1` e `3`). O
contrato de saída (nome do QVD, campos) não mudou, então a Modelagem
não precisou de nenhum ajuste.

**2026-08-27 (5ª leva) — bug corrigido, achado durante validação pelo
usuário:** RCA `6026` aparecia com `PercAtingimentoFaturado` somando
**2** em vez de 1 para o indicador MIX MINIMO (sinal de duplicidade).
Causa raiz confirmada direto na planilha: **`COD_GRUPO` na aba
`MIXMIN_GRUPO_PRODUTO` não é um id único global — ele se repete por
Categoria** (Grupo 1 existe em DPP, HFS, C&C e NRM, cada um um produto
diferente). 258 dos 332 produtos estão associados a mais de um par
(Grupo, Categoria) — ex: produto `219481` é `(Grupo 1, DPP)` e também
`(Grupo 1, HFS)`. `TRF_MIXMIN_PRODUTO_GRUPO` não carregava `Categoria`,
então o cruzamento de vendas→grupo (seção 7.5) batia em **todos** os
grupos com aquele número, de categorias não relacionadas, inflando a
quantidade comprada. Corrigido: `TRF_MIXMIN_PRODUTO_GRUPO` agora carrega
`Categoria`; criado `MAP_CLIENTE_CATEGORIA` (Cliente+Filial→Categoria,
a partir de `TRF_MIX_MIN`) para trazer a categoria do próprio cliente
para `TEMP_VENDAS_MIXMIN`; o cruzamento agora usa a chave composta
`CodProduto + Categoria`.

> **Ajuste adicional no mesmo dia, a pedido do usuário**: mesmo com
> `Categoria` no cruzamento, agregar por `CodGrupo` ainda dependia de um
> código numérico que **não é o identificador de negócio real do grupo**
> — confirmado na planilha que o mesmo `NOMEGRUPO` pode ter `CodGrupo`
> diferente em categorias diferentes (103 dos 186 nomes de grupo caem
> nesse caso). Como `NOMEGRUPO` já vem preenchido em toda linha de
> produto da aba (não só na primeira linha do grupo), a agregação inteira
> foi trocada de `CodGrupo` para `NomeGrupo` — `TRF_MIXMIN_PRODUTO_GRUPO`
> e `TRF_MIXMIN_GRUPO` não carregam mais `COD_GRUPO`, e todas as chaves
> de join/agregação da seção 7 (7.4 `MIXMIN_ELEGIVEL`, 7.5
> `COMPRA_CLIENTE_GRUPO`, 7.6 `LEFT JOIN`) usam `NomeGrupo` no lugar de
> `CodGrupo`. Confirmado que, dentro da mesma Categoria, `CodGrupo` e
> `NomeGrupo` já eram 1:1 (0 inconsistências), então essa troca não muda
> o resultado matemático da versão anterior — só remove a dependência de
> um código cuja unicidade só valia por acidente do agrupamento por
> Categoria, deixando o identificador de negócio real como chave.
> **Ainda não validado no Qlik Sense após esta correção.**

> **Segundo problema encontrado durante a mesma investigação, NÃO
> corrigido no script** (usuário optou por corrigir na planilha): a
> categoria de cliente aparece grafada como `NMR` na aba `MIX_MIN` mas
> como `NRM` na aba `MIXMIN_GRUPO_PRODUTO` (letras trocadas). Como a
> seção 7.4 faz um `JOIN` (inner join) por `Categoria`, clientes com
> categoria `NMR` não encontram nenhum grupo correspondente e **somem
> inteiramente** de `MIXMIN_ELEGIVEL` — o que pode fazer o RCA inteiro
> sumir do indicador Mix Mínimo (se todos os clientes dele forem `NMR`)
> ou dar resultado incorreto (se o RCA tiver clientes mistos, o cliente
> `NMR` é ignorado na regra tudo-ou-nada). **Ação: corrigir a grafia na
> planilha** (unificar `NMR`/`NRM` para o mesmo valor nas duas abas)
> antes da próxima carga.

**2026-08-27 (6ª leva) — bug corrigido, achado após o usuário confirmar
que o "2" persistia mesmo depois da correção acima e do ajuste
NMR/NRM na planilha:** o problema não estava mais no cálculo do
Mix Mínimo em si (dimensão simples já mostrava `PercAtingimentoFaturado
= 1` corretamente para o RCA `6026`) — era uma **referência circular na
Modelagem**, visível no diagrama do modelo de dados do Qlik: `Sum()`
sobre o campo duplicava o valor, mas o campo como dimensão não mudava
(sintoma clássico de loop de associação). Duas causas encontradas na
seção "PREMIAÇÃO GILLETTE TRIMESTRAL":
1. **`BASE_GILLETTE_TRI` (que vira `PREMIACAO_GILLETTE_TRI`) carregava
   `CodSupervisor` e `PercAtingimentoFaturado`** com os mesmos nomes que
   já existem em `BASE_RCA_INDICADORES_REALIZADO`** — como as duas
   tabelas permanecem no modelo final, isso as ligava por **três campos
   simultâneos** (`CodRca` + `CodSupervisor` + `PercAtingimentoFaturado`),
   o que o Qlik também trata como referência circular entre duas
   tabelas. **Corrigido**: renomeados para `CodSupervisorGillette` e
   `PercAtingimentoFaturadoGillette` dentro de `BASE_GILLETTE_TRI`
   (e a referência em `RANKING_GILLETTE` atualizada) — agora `CodRca` é
   o único campo em comum entre as duas tabelas, como deveria ser.
2. **`BASE_GILLETTE_TRI` também copiava (via `ApplyMap`) o valor de
   devolução de `GANHO_FINAL_RCA` usando o MESMO nome de campo**
   `PercDevolucaoRcaFaturado` que já existe em `GANHO_FINAL_RCA` — isso
   ligava `GANHO_FINAL_RCA` e `PREMIACAO_GILLETTE_TRI` diretamente, e
   como as duas também se ligam a `BASE_RCA_INDICADORES_REALIZADO` via
   `CodRca`, fechava um loop de 3 tabelas (exatamente o mostrado no
   diagrama do modelo de dados que o usuário enviou).

   **Correção original errada (2026-08-27, corrigida na hora):** cheguei
   a colocar um `DROP TABLE GANHO_FINAL_RCA;` achando que era uma tabela
   intermediária esquecida — **estava errado**: `GANHO_FINAL_RCA` é a
   base de premiação final do RCA, usada pelo usuário para validar
   contra o total de devolução dele, e precisa continuar no modelo.
   Revertido o `DROP` imediatamente após o usuário apontar o problema.

   **Correção final**: renomeada a cópia dentro de `BASE_GILLETTE_TRI`
   para `PercDevolucaoRcaFaturadoGillette` (só usada internamente para o
   cálculo do ranking Gillette) — `GANHO_FINAL_RCA` continua intacta no
   modelo, com todos os seus campos (incluindo o `PercDevolucaoRcaFaturado`
   original), ligada a `BASE_RCA_INDICADORES_REALIZADO` e a
   `PREMIACAO_GILLETTE_TRI` **só por `CodRca`** (formato estrela, sem
   ciclo). **Ainda não validado no Qlik Sense.**

## 7. Pontos em aberto para continuar o projeto

- Confirmar onde vivem os indicadores **PET** e os supervisores
  fictícios **Gerson (7374)** / **Suzy (240340)** — não estão neste
  `.QVS`.
- Confirmar se o valor de R$20 por positivação do **Escolha Certa
  Especial** é calculado em algum lugar (script ou app Qlik) — não
  localizado aqui.
- **Corrigir na planilha** a grafia divergente `NMR` (aba `MIX_MIN`) vs
  `NRM` (aba `MIXMIN_GRUPO_PRODUTO`) — enquanto não for corrigido,
  clientes dessa categoria ficam de fora do cálculo de Mix Mínimo.
- Validar no Qlik Sense as três correções do bug de duplicidade do **Mix
  Mínimo** (cruzamento Produto+Categoria na seção 4, e as duas
  referências circulares na Modelagem/Gillette) — conferir que
  `Sum([PercAtingimentoFaturado])` não soma mais valores > 1 num gráfico
  (ex: RCA 6026) e que o diagrama do modelo de dados mostra
  `GANHO_FINAL_RCA` **ainda presente** (não deve sumir - é a base de
  premiação final do RCA), mas ligada a `BASE_RCA_INDICADORES_REALIZADO`
  e `PREMIACAO_GILLETTE_TRI` só por `CodRca`, sem loop entre as 3.
- **PREM_RANK_GILLETTE_SUP** foi só transformado (extração), ainda sem
  nenhum cálculo de premiação por Supervisor ligado a ele.
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
