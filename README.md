# FieldOps — Análise de Telemetria GPS por Área Agrícola

Notebook do **Google Colab** que identifica em qual área de um arquivo **KML**
(fazendas, talhões, setores ou blocos) cada ponto GPS de telemetria ocorreu e
calcula **quanto tempo cada equipamento trabalhou dentro de cada área**.

➡️ **Notebook:** [`FieldOps_GPS_Colab.ipynb`](FieldOps_GPS_Colab.ipynb)

## Como usar

1. Abra o `FieldOps_GPS_Colab.ipynb` no [Google Colab](https://colab.research.google.com/)
   (`File → Upload notebook` ou abra direto pelo GitHub).
2. Execute as células **na ordem** (`Runtime → Run all`).
3. Quando solicitado, faça o **upload** de:
   - um arquivo **`.kml`** com os polígonos das áreas;
   - um arquivo **`.zip`** contendo um ou mais CSVs de telemetria.

Não é necessário instalar nada localmente — a primeira célula instala todas as
dependências no próprio Colab.

## Entradas

### KML
Polígonos das áreas agrícolas. Todos os atributos (nome, descrição e campos de
`ExtendedData`) são preservados e anexados a cada ponto.

### ZIP (telemetria)
Um ou mais CSVs com as colunas:

```
ceqid, nickname, vin, name, numeric_value, text_value, uom, event_timestamp, lat, lon
```

Colunas obrigatórias: `ceqid`, `lat`, `lon`, `event_timestamp`.

## Saídas

| Arquivo | Conteúdo |
|---|---|
| `resultado_pontos.xlsx` / `.csv` | Todos os pontos com a área associada e o tempo calculado (exportado em **CSV** quando ultrapassa o limite de 1.048.576 linhas do Excel) |
| `resumo_por_area.xlsx` | Pontos, horas trabalhadas, **horas efetivas**, **horas com piloto**, equipamentos e primeiro/último registro por área |
| `resumo_por_equipamento.xlsx` | Horas trabalhadas, **horas efetivas**, **horas com piloto** e pontos por equipamento e área |
| `mapa.html` | Mapa interativo (Folium) com polígonos e pontos GPS |

Todos são disponibilizados para download automaticamente.

## Estados de operação (a partir da coluna `name`)

A telemetria vem em **formato longo** (cada linha é uma leitura de um sinal
identificado pela coluna `name`). O notebook reconstrói uma tabela com uma linha
por amostra e calcula dois estados:

| Estado | Regra |
|---|---|
| **Tempo efetivo** | carga do motor **≥ 35%** **E** status do elevador = `forward` |
| **Piloto ligado** | "Status da orientação automática" = `on` |

As palavras-chave que identificam cada sinal são configuráveis no topo do
notebook (`SINAL_CARGA_MOTOR`, `SINAL_STATUS_ELEVADOR`, `SINAL_PILOTO`,
`LIMIAR_CARGA_MOTOR`, etc.). A correspondência ignora acentos e maiúsculas, e o
notebook corrige automaticamente acentuação quebrada (mojibake, ex.:
`orientaÃ§Ã£o` → `orientação`).

## Regras de cálculo de tempo

- Ordena por `ceqid` e `event_timestamp`.
- Tempo de cada ponto = diferença para o ponto anterior **do mesmo equipamento**.
- Intervalos **negativos** ou **maiores que 10 minutos** são ignorados.
- Os sinais de estado (elevador, piloto, carga) recebem *forward-fill* por
  equipamento — o último valor conhecido persiste até a próxima mudança.
- A coluna **`data`** do resumo por equipamento e os timestamps exportados usam o
  **fuso local** (`TIMEZONE_LOCAL`, padrão `America/Sao_Paulo`), de modo que o
  "dia" respeite o horário de Brasília. Ajuste `TIMEZONE_LOCAL` no topo do
  notebook se sua operação usar outro fuso.

## Desempenho

Projetado para **centenas de milhares de pontos**: usa GeoPandas Spatial Join
com **índice espacial (R-tree)** e operações **vetorizadas**, evitando loops
linha a linha.
