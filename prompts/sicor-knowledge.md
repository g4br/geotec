# 🗄️ SICOR/PROAGRO SQL KNOWLEDGE BASE

## System Prompt para Geração de Consultas SQL

> **PROPÓSITO**: Este documento serve como contexto inicial (Knowledge Base) para qualquer LLM gerar consultas SQL corretas a partir de linguagem natural, no contexto do banco de dados SICOR (Sistema de Operações do Crédito Rural) e Proagro (Programa de Garantia da Atividade Agropecuária).

---

## 📋 SUMÁRIO

1. [Visão Geral do Banco de Dados](#1-visão-geral-do-banco-de-dados)
2. [Conceitos Fundamentais](#2-conceitos-fundamentais)
3. [Catálogo de Tabelas](#3-catálogo-de-tabelas)
4. [Relacionamentos entre Tabelas](#4-relacionamentos-entre-tabelas)
5. [Padrões de Consulta](#5-padrões-de-consulta)
6. [Funções PostGIS](#6-funções-postgis)
7. [Exemplos Práticos](#7-exemplos-práticos)
8. [Armadilhas Comuns](#8-armadilhas-comuns)
9. [Templates de Consulta](#9-templates-de-consulta)

---

## 1. VISÃO GERAL DO BANCO DE DADOS

### 1.1 Contexto de Negócio

O banco de dados contém informações sobre:

- **Crédito Rural**: Operações de financiamento agrícola no Brasil
- **Proagro**: Programa de seguro agrícola governamental que cobre perdas por eventos climáticos adversos
- **ZARC**: Zoneamento Agrícola de Risco Climático - define janelas de plantio por cultura/município
- **Glebas**: Parcelas georreferenciadas de terra vinculadas às operações
- **Dados Fundiários**: CAR, Assentamentos, Terras Indígenas, Quilombolas, Embargos

### 1.2 SGBD e Extensões

- **PostgreSQL** com extensão **PostGIS** para dados geoespaciais
- Sistema de Referência Espacial: **EPSG:4674** (SIRGAS 2000)
- Geometrias armazenadas em formato WKT ou tipo `geometry`

### 1.3 Chave Principal do Sistema

A maioria das tabelas do SICOR se relaciona através da **chave composta**:
- `ref_bacen` (INTEGER): Número mascarado de referência do contrato
- `nu_ordem` (INTEGER): Número da destinação/finalidade dentro do contrato

> **IMPORTANTE**: Um contrato (`ref_bacen`) pode ter múltiplas destinações (`nu_ordem`), por exemplo: custeio de soja + aquisição de trator.

---

## 2. CONCEITOS FUNDAMENTAIS

### 2.1 Hierarquia do Crédito Rural

```
CONTRATO (ref_bacen)
├── DESTINAÇÃO 1 (nu_ordem = 1) → Empreendimento: Custeio Soja
│   ├── Glebas (parcelas georreferenciadas)
│   ├── Pedido de Cobertura Proagro (COP)
│   └── Parcelas de pagamento
└── DESTINAÇÃO 2 (nu_ordem = 2) → Empreendimento: Aquisição Trator
    └── Parcelas de pagamento
```

### 2.2 Fluxo do Proagro

1. **Contratação**: Produtor contrata financiamento com cobertura Proagro
2. **Comunicação de Perda (COP)**: Ocorre evento adverso → produtor comunica
3. **Comprovação (RCP)**: Perito realiza vistoria e emite relatório
4. **Julgamento**: Instituição financeira julga o pedido
5. **Pagamento**: Se deferido, parcelas são pagas

### 2.3 Código do Empreendimento (14 dígitos)

O `cd_empreendimento` segue estrutura específica:
- **Dígito 1**: Atividade (1=agrícola, 2=pecuária)
- **Dígito 2**: Finalidade (1=comercialização, 2=custeio, 3=investimento, 4=industrialização)
- **Dígitos 3-4**: Modalidade (01=lavoura, 60=aquicultura, etc.)
- **Dígitos 5-8**: Produto (0067=soja, 0044=milho, etc.)
- **Dígitos 9-11**: Variedade
- **Dígito 12**: Consórcio
- **Dígito 13**: Cesta de safra
- **Dígito 14**: Zoneamento

### 2.4 ZARC - Decêndios

O ano é dividido em 36 **decêndios** (períodos de ~10 dias):
- Decêndio 1: 01-10 de Janeiro
- Decêndio 2: 11-20 de Janeiro
- Decêndio 3: 21-31 de Janeiro
- ... e assim por diante

O ZARC indica `risco = 0` para decêndios com baixo risco de plantio (recomendados).

---

## 3. CATÁLOGO DE TABELAS

### 3.1 TABELAS PRINCIPAIS (Operações e Proagro)

#### `sicor_operacao_basica_estado`
**Descrição**: Tabela central com todas as operações de crédito rural contratadas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK** - Número mascarado do contrato |
| `nu_ordem` | INTEGER | **PK** - Número da destinação |
| `cnpj_if` | CHAR(8) | FK → ifssicor (instituição financeira) |
| `dt_emissao` | DATE | Data de emissão do contrato |
| `dt_vencimento` | DATE | Data de vencimento |
| `cd_inst_credito` | INTEGER | FK → instrumentocredito |
| `cd_categ_emitente` | CHAR(4) | FK → categoriaemitente (pequeno/médio/grande) |
| `cd_fonte_recurso` | CHAR(4) | FK → fonterecursos |
| `cd_tipo_seguro` | CHAR(1) | FK → tipogarantiaempreendimento ('1'=Proagro, '2'=Proagro Mais) |
| `cd_empreendimento` | CHAR(14) | FK → empreendimento |
| `cd_programa` | CHAR(4) | FK → programa |
| `cd_estado` | CHAR(2) | UF do contrato |
| `vl_parc_credito` | NUMERIC | Valor da parcela de crédito |
| `vl_area_financ` | NUMERIC | Área financiada (hectares) |
| `vl_prev_prod` | NUMERIC | Produção prevista |
| `dt_inic_plantio` | DATE | Data início do plantio |
| `dt_fim_plantio` | DATE | Data fim do plantio |
| `dt_inic_colheita` | DATE | Data início da colheita |
| `dt_fim_colheita` | DATE | Data fim da colheita |
| `cd_ciclo_cultivar` | INTEGER | FK → ciclocultivarproagro |
| `cd_tipo_solo` | INTEGER | FK → tiposoloproagro |
| `cd_tipo_agricultura` | CHAR(1) | FK → tipoagropecuaria |
| `cd_tipo_irrigacao` | CHAR(1) | FK → tipoirrigacao |
| `cd_tipo_cultivo` | CHAR(2) | FK → tipocultivo |
| `recurso_publico` | BOOLEAN | Se usa recurso público |

---

#### `sicor_cop_basico`
**Descrição**: Pedidos de Cobertura do Proagro (Comunicação de Perdas).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK → sicor_operacao_basica_estado |
| `nu_ordem` | INTEGER | **PK**, FK → sicor_operacao_basica_estado |
| `cd_evento` | INTEGER | **PK**, FK → eventoproagro (tipo do evento adverso) |
| `dt_comunicacao` | DATE | Data da comunicação da perda |
| `cd_status` | INTEGER | FK → statuscopproagro |
| `dt_inicio_plantio` | DATE | Data início plantio informada na COP |
| `dt_fim_plantio` | DATE | Data fim plantio informada na COP |
| `dt_inicio_colheita` | DATE | Data início colheita |
| `dt_fim_colheita` | DATE | Data fim colheita |
| `cd_ciclo_cultivar` | INTEGER | FK → ciclocultivarproagro |
| `cd_tipo_solo` | INTEGER | FK → tiposoloproagro |

---

#### `sicor_rcp_basico`
**Descrição**: Relatório de Comprovação de Perdas (vistoria pericial).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `cd_evento` | INTEGER | Evento verificado pelo perito |
| `cd_tipo` | INTEGER | Tipo de RCP |
| `cd_status` | INTEGER | Status do RCP |
| `dt_visita` | DATE | Data da visita do perito |
| `dt_entrega` | DATE | Data de entrega do relatório |
| `dt_inicio_evento` | DATE | Início do evento adverso |
| `dt_fim_evento` | DATE | Fim do evento adverso |
| `vl_area` | NUMERIC | Área verificada |
| `vl_prev_prod` | NUMERIC | Produção prevista original |
| `vl_rec_prev` | NUMERIC | Receita prevista |
| `nu_dias_ciclo_cultivar` | INTEGER | Dias do ciclo da cultivar |

---

#### `sicor_parcelas_proagro`
**Descrição**: Parcelas de pagamento do Proagro.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `cd_natureza_parcela` | CHAR(3) | FK → naturezaproagro |
| `cd_status` | INTEGER | FK → statusparcelaproagro |
| `cd_instancia` | INTEGER | FK → instanciaproagro |
| `vl_base` | NUMERIC | Valor base |
| `vl_atual` | NUMERIC | Valor atual |
| `vl_pago` | NUMERIC | Valor efetivamente pago |
| `dt_pagamento` | DATE | Data do pagamento |
| `dt_base` | DATE | Data base |
| `dt_atualizacao` | DATE | Data de atualização |
| `dt_remessa` | DATE | Data de remessa |

---

#### `sicor_sumula_julgamento_basico`
**Descrição**: Súmula do julgamento do pedido Proagro.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `cd_decisao` | INTEGER | Código da decisão |
| `cd_status` | INTEGER | Status da súmula |
| `cd_instancia` | INTEGER | FK → instanciaproagro |
| `dt_decisao` | DATE | Data da decisão |
| `dt_inclusao` | DATE | Data de inclusão |
| `vl_receitas_consideradas` | NUMERIC | Receitas consideradas |
| `vl_perdas_nao_amparadas` | NUMERIC | Perdas não amparadas |
| `vl_cobertura_ant_credito_custeio` | NUMERIC | Cobertura anterior do crédito |
| `vl_cred_custeio_usado` | NUMERIC | Crédito de custeio utilizado |
| `vl_rec_prop_usado` | NUMERIC | Recursos próprios utilizados |
| `vl_perc_redutor_cobertura` | NUMERIC | Percentual redutor |
| `nu_dias_uteis_atraso_perito` | INTEGER | Dias de atraso do perito |
| `ib_segunda_vistoria` | INTEGER | Indicador de segunda vistoria |

---

### 3.2 TABELAS DE GLEBAS (Geoespaciais)

#### `sicor_glebas_wkt`
**Descrição**: Glebas em formato texto WKT (original do SICOR).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `nu_indice` | INTEGER | **PK** - Identificador da gleba na operação |
| `gt_geometria` | TEXT | Geometria em formato WKT (SIRGAS 2000) |

#### `sicor_glebas`
**Descrição**: Glebas processadas com coluna geometry (derivada).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | BIGINT | **PK** - ID sequencial |
| `ref_bacen` | INTEGER | FK |
| `nu_ordem` | INTEGER | FK |
| `nu_indice` | INTEGER | Identificador da gleba |
| `data_emissao_contrato` | DATE | Data de emissão |
| `geom` | GEOMETRY(GEOMETRY, 4674) | Geometria PostGIS |
| `area_gleba` | NUMERIC | Área calculada (m²) |
| `perimetro_gleba` | NUMERIC | Perímetro (m) |
| `area_menor_retangulo_envolvente` | NUMERIC | Área do MBR |
| `area_menor_circulo_envolvente` | NUMERIC | Área do círculo envolvente |

#### `sicor_rcp_glebas` / `sicor_rcp_glebas_wkt`
**Descrição**: Glebas dos Relatórios de Comprovação de Perdas (estrutura similar).

---

### 3.3 TABELAS AUXILIARES (Lookup/Domínio)

#### `empreendimento`
**Descrição**: Cadastro de empreendimentos financiáveis.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_empreendimento` | CHAR(14) | **PK** - Código único |
| `atividade` | TEXT | 'agrícola' ou 'pecuária' |
| `finalidade` | TEXT | 'custeio', 'investimento', 'comercialização', 'industrialização' |
| `modalidade` | TEXT | 'lavoura', 'pastagem', 'bovinocultura', etc. |
| `produto` | TEXT | 'soja', 'milho', 'café', 'trator', etc. |
| `variedade` | TEXT | Variedade do produto |
| `cesta` | TEXT | 'safra de verão', 'safrinha', etc. |
| `zoneamento` | TEXT | Informação de zoneamento |
| `unidade_medida` | TEXT | 'hectare', 'cabeça', etc. |
| `unidade_medida_previsao` | TEXT | 'tonelada', 'arroba', etc. |
| `consorcio` | TEXT | Informação de consórcio |
| `data_inicio` | DATE | Início de validade |
| `data_fim` | DATE | Fim de validade |

---

#### `eventoproagro`
**Descrição**: Tipos de eventos adversos do Proagro.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_evento` | INTEGER | **PK** |
| `nome_evento` | TEXT | Nome do evento |

**Valores comuns**:
- 17: Chuva excessiva
- 24: Geada
- 31: Granizo
- 48: Seca
- 55: Tromba de água
- 61: Vendaval
- 79: Vento forte
- 86: Variação excessiva de temperatura
- 93: Raio
- 103: Outros fenômenos naturais fortuitos
- 110: Doença ou praga
- 127: Enchentes
- 135: Chuva na colheita

---

#### `statuscopproagro`
**Descrição**: Status dos pedidos de cobertura.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_status` | INTEGER | **PK** |
| `descricao` | TEXT | Descrição do status |

---

#### `categoriaemitente`
**Descrição**: Categoria do produtor rural.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_categ_emitente` | CHAR(4) | **PK** |
| `descricao` | TEXT | 'Pequeno produtor', 'Médio produtor', 'Grande produtor' |
| `valor_limite` | NUMERIC | Limite de valor |
| `area_maxima` | NUMERIC | Área máxima |

---

#### `tipogarantiaempreendimento`
**Descrição**: Tipos de garantia/seguro.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_seguro` | CHAR(1) | **PK** |
| `descricao` | TEXT | Descrição |

**Valores**:
- '0': Sem seguro
- '1': Proagro
- '2': Proagro Mais
- '3': Seguro privado

---

#### `programa`
**Descrição**: Programas de crédito rural (PRONAF, etc.).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_programa` | CHAR(4) | **PK** |
| `descricao` | TEXT | Nome do programa |
| `financiamento` | TEXT | Tipo de financiamento |

---

#### `fonterecursos` / `fonterecursospublicos`
**Descrição**: Fontes de recursos dos financiamentos.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_fonte_recurso` | CHAR(4) | **PK** |
| `descricao` | TEXT | Descrição da fonte |

---

#### `ifssicor`
**Descrição**: Instituições Financeiras.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cnpj_if` | CHAR(8) | **PK** - CNPJ base |
| `nome_if` | TEXT | Nome da instituição |
| `segmento_if` | TEXT | Segmento (banco, cooperativa, etc.) |

---

#### `ciclocultivarproagro`
**Descrição**: Ciclos de cultivar para Proagro/ZARC.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_ciclo_cultivar` | INTEGER | **PK** |
| `descricao_ciclo` | TEXT | 'Grupo I', 'Grupo II', 'Grupo III' |

---

#### `tiposoloproagro`
**Descrição**: Tipos de solo para Proagro/ZARC.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_solo` | INTEGER | **PK** |
| `descricao_tipo_solo` | TEXT | 'Tipo 1', 'Tipo 2', 'Tipo 3' (capacidade de armazenamento de água) |

---

#### `instrumentocredito`
**Descrição**: Tipos de instrumento de crédito.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_inst_credito` | INTEGER | **PK** |
| `sigla` | TEXT | Sigla |
| `descricao` | TEXT | 'Cédula de Crédito Bancário', 'NPR', etc. |

---

#### `naturezaproagro`
**Descrição**: Natureza das parcelas do Proagro.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_natureza_parcela` | CHAR(3) | **PK** |
| `descricao` | TEXT | Descrição |
| `sinal` | NUMERIC | 1.0 para crédito, -1.0 para devolução |

---

#### `instanciaproagro`
**Descrição**: Instâncias de julgamento.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_instancia` | INTEGER | **PK** |
| `descricao` | TEXT | '1ª Instância', '2ª Instância', 'BCB' |

---

### 3.4 TABELA DO ZARC

#### `zarc`
**Descrição**: Zoneamento Agrícola de Risco Climático (formato expandido).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` | INTEGER | **PK** |
| `geocodigo` | INTEGER | Código IBGE do município |
| `municipio` | TEXT | Nome do município |
| `uf` | CHAR(2) | UF |
| `cultura` | TEXT | Nome da cultura |
| `cod_cultura` | CHAR(4) | Código da cultura |
| `grupo` | TEXT | Grupo de ciclo ('Grupo I', 'Grupo II', 'Grupo III') |
| `solo` | TEXT | Tipo de solo |
| `cod_solo` | INTEGER | Código do solo (1, 2, 3) |
| `safra` | TEXT | Safra de referência |
| `decendio` | INTEGER | Número do decêndio (1-36) |
| `risco` | INTEGER | **0 = baixo risco (recomendado)**, 1 = alto risco |
| `data_inicial_decendio` | DATE | Data inicial do decêndio |
| `data_final_decendio` | DATE | Data final do decêndio |
| `portaria` | TEXT | Número da portaria MAPA |
| `clima` | TEXT | Tipo de clima |

#### `zarc_oficial` / `zarc_oficial_completo`
**Descrição**: Formato pivotado com decêndios em colunas (decendio_1 a decendio_36).

---

### 3.5 TABELAS DE COMPLEMENTO

#### `sicor_complemento_operacao_basica`
**Descrição**: Dados complementares das operações.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `ref_bacen_efetivo` | TEXT | Ref_bacen não mascarado |
| `agencia_if` | TEXT | Código da agência |
| `cd_ibge_municipio` | INTEGER | Código IBGE do município |
| `num_cedula_if` | TEXT | Número da cédula |

#### `sicor_complemento_cop`
**Descrição**: Complemento dos pedidos de cobertura (dados do perito).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `cd_evento` | INTEGER | **PK**, FK |
| `cd_cpf_perito` | TEXT | CPF do perito (mascarado) |
| `cd_cpf_cnpj_periciadora` | TEXT | CPF/CNPJ da empresa periciadora |

---

### 3.6 TABELAS DE MUTUÁRIOS E PROPRIEDADES

#### `sicor_mutuarios`
**Descrição**: Beneficiários dos contratos.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | FK |
| `cd_cpf_cnpj` | TEXT | CPF/CNPJ (mascarado) |
| `cd_tipo_beneficiario` | INTEGER | FK → tipobeneficiario |
| `cd_sexo` | INTEGER | 1=Mulher, 2=Homem |
| `cd_dap` | TEXT | Código DAP (agricultura familiar) |
| `cd_primeiro` | CHAR(1) | Se é o primeiro mutuário |

#### `sicor_propriedades`
**Descrição**: Propriedades rurais vinculadas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `cd_cnpj_cpf` | TEXT | CPF/CNPJ do proprietário |
| `cd_nirf` | TEXT | Número do NIRF |
| `cd_car` | TEXT | Código do CAR |
| `cd_sncr` | TEXT | Código SNCR |

---

### 3.7 TABELAS GEOGRÁFICAS AUXILIARES

#### `municipios_2022`
**Descrição**: Municípios brasileiros com geometria.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `cd_mun` | INTEGER | Código IBGE |
| `nm_mun` | TEXT | Nome do município |
| `sigla_uf` | CHAR(2) | UF |
| `area_km2` | NUMERIC | Área em km² |
| `geom` | GEOMETRY | Geometria do município |
| `num_contratos` | INTEGER | Contratos (pré-calculado) |
| `valor_financiamento` | NUMERIC | Valor total (pré-calculado) |
| `num_pedidos_proagro` | INTEGER | Total de pedidos Proagro |
| `num_pedidos_deferidos_proagro` | INTEGER | Pedidos deferidos |
| `valor_pago_proagro` | NUMERIC | Valor pago Proagro |

#### `estados_2022`
**Descrição**: Estados brasileiros com geometria.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `cd_uf` | CHAR(2) | Código UF |
| `sigla_uf` | CHAR(2) | Sigla |
| `nm_uf` | TEXT | Nome |
| `nm_regiao` | TEXT | Região |
| `area_km2` | NUMERIC | Área |
| `geom` | GEOMETRY | Geometria |

#### `biomas`
**Descrição**: Biomas brasileiros.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `bioma` | TEXT | Nome do bioma |
| `cd_bioma` | INTEGER | Código |
| `geom` | GEOMETRY | Geometria |

---

### 3.8 TABELAS DE RESTRIÇÕES AMBIENTAIS E FUNDIÁRIAS

#### `car_area_imovel`
**Descrição**: Área do imóvel no CAR.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `cod_imovel` | TEXT | Código do imóvel CAR |
| `cod_imovel_sicor` | TEXT | Código para vínculo com SICOR |
| `num_area` | NUMERIC | Área em hectares |
| `mod_fiscal` | NUMERIC | Módulos fiscais |
| `municipio` | TEXT | Município |
| `cod_estado` | CHAR(2) | UF |
| `geom` | GEOMETRY | Geometria |

#### Outras tabelas CAR:
- `car_app_*`: Áreas de Preservação Permanente
- `car_reserva_legal_*`: Reserva Legal
- `car_vegetacao_nativa_*`: Vegetação Nativa
- `car_area_consolidada_*`: Área Consolidada
- `car_uso_restrito_*`: Uso Restrito

#### `embargo_ibama_*`
**Descrição**: Áreas embargadas pelo IBAMA.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `num_auto_i` | TEXT | Número do auto de infração |
| `data_embargo` / `dat_embarg` | DATE | Data do embargo |
| `des_infrac` | TEXT | Descrição da infração |
| `cpf_cnpj_*` | TEXT | CPF/CNPJ do autuado |
| `geom` | GEOMETRY | Área embargada |

#### `embargos_icmbio`
**Descrição**: Embargos do ICMBio em Unidades de Conservação.

#### `terras_indigenas` / `tis_poligonais`
**Descrição**: Terras indígenas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `terrai_nom` | TEXT | Nome da TI |
| `fase_ti` | TEXT | Fase de regularização |
| `etnia_nome` | TEXT | Etnia |
| `superficie` | NUMERIC | Área |
| `geom` | GEOMETRY | Geometria |

#### `areas_quilombolas`
**Descrição**: Comunidades quilombolas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `nm_comunid` | TEXT | Nome da comunidade |
| `fase` | TEXT | Fase de regularização |
| `nr_area_ha` | NUMERIC | Área em hectares |
| `geom` | GEOMETRY | Geometria |

#### `assentamentos`
**Descrição**: Assentamentos de reforma agrária.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `nome_proje` | TEXT | Nome do projeto |
| `capacidade` | INTEGER | Capacidade de famílias |
| `num_famili` | INTEGER | Número de famílias |
| `geom` | GEOMETRY | Geometria |

#### `cnuc_2024_02`
**Descrição**: Cadastro Nacional de Unidades de Conservação.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `nome_uc` | TEXT | Nome da UC |
| `categoria` | TEXT | Categoria (Parque, RESEX, etc.) |
| `esfera` | TEXT | Federal, Estadual, Municipal |
| `area_ha` | NUMERIC | Área em hectares |
| `geom` | GEOMETRY | Geometria |

---

### 3.9 TABELAS AUXILIARES ADICIONAIS (Lookup/Domínio)

#### `tipobeneficiario`
**Descrição**: Tipos de beneficiário do crédito rural.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_beneficiario` | INTEGER | **PK** |
| `descricao` | TEXT | 'Produtor Rural PF', 'Cooperativa', 'Agroindústria', etc. |

---

#### `tipoagropecuaria`
**Descrição**: Tipo de atividade agropecuária.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_agricultura` | CHAR(1) | **PK** |
| `descricao` | TEXT | 'Convencional', 'Orgânica', 'Transgênica', etc. |

---

#### `tipocultivo`
**Descrição**: Tipos de cultivo.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_cultivo` | CHAR(2) | **PK** |
| `descricao` | TEXT | 'Cultivo Mínimo', 'Plantio Direto', etc. |

---

#### `tipoirrigacao`
**Descrição**: Tipos de irrigação.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_irrigacao` | CHAR(1) | **PK** |
| `descricao` | TEXT | 'Não Irrigado', 'Irrigado', 'Sequeiro', etc. |

---

#### `tipointegracao`
**Descrição**: Tipos de integração/consórcio.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_intgr_consor` | CHAR(1) | **PK** |
| `descricao` | TEXT | 'Não consorciado', 'ILP', 'ILPF', etc. |

---

#### `subprograma`
**Descrição**: Subprogramas de crédito rural (vinculados a programas).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_subprograma` | CHAR(4) | **PK** |
| `codigo_programa` | CHAR(4) | FK → programa |
| `descricao_subprograma` | TEXT | Descrição do subprograma |
| `vl_taxa_juros` | NUMERIC | Taxa de juros |

---

#### `fasecicloproducao`
**Descrição**: Fases do ciclo de produção.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_fase_ciclo_producao` | CHAR(1) | **PK** |
| `descricao` | TEXT | 'Pré-plantio', 'Plantio', 'Colheita', etc. |

---

#### `graosemente`
**Descrição**: Tipos de grão/semente.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_grao_semente` | CHAR(1) | **PK** |
| `descricao` | TEXT | 'Grão', 'Semente', etc. |

---

#### `situacaooperacao`
**Descrição**: Situação da operação de crédito.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_situacao_operacao` | INTEGER | **PK** |
| `descricao` | TEXT | 'Normal', 'Liquidada', 'Prorrogada', etc. |

---

#### `motivodesclassificacao`
**Descrição**: Motivos de desclassificação de operações.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_motivo_desc` | INTEGER | **PK** |
| `descricao` | TEXT | Descrição do motivo |

---

#### `statusparcelaproagro`
**Descrição**: Status das parcelas do Proagro.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_status` | INTEGER | **PK** |
| `descricao` | TEXT | 'Pendente', 'Paga', 'Cancelada', etc. |

---

#### `encargosfinanceiroscomplementares`
**Descrição**: Tipos de encargos financeiros complementares.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `cd_tipo_encarg_financ` | INTEGER | **PK** |
| `descricao` | TEXT | Descrição do encargo |

---

### 3.10 TABELAS SICOR ADICIONAIS

#### `sicor_desclassificacao`
**Descrição**: Desclassificações de operações (irregularidades identificadas).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `dt_desc` | DATE | Data da desclassificação |
| `cd_motivo_desc` | INTEGER | FK → motivodesclassificacao |
| `vl_desc` | NUMERIC | Valor desclassificado |
| `tipo_desc` | TEXT | Tipo da desclassificação |

---

#### `sicor_complemento_rcp`
**Descrição**: Complemento do Relatório de Comprovação de Perdas (dados do perito).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `cd_cpf_perito` | TEXT | CPF do perito (mascarado) |
| `cd_cpf_cnpj_periciadora` | TEXT | CPF/CNPJ da empresa periciadora |

---

#### `sicor_liberacao_recursos`
**Descrição**: Liberações de recursos das operações.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `dt_liberacao` | DATE | Data da liberação |
| `vl_liberado` | NUMERIC | Valor liberado |

---

#### `sicor_parcelas_desembolso`
**Descrição**: Cronograma de desembolso das operações.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | **PK**, FK |
| `nu_ordem` | INTEGER | **PK**, FK |
| `dt_prev_pagamento` | DATE | Data prevista de pagamento |
| `valor_parcela` | NUMERIC | Valor da parcela |

---

#### `sicor_lista_cooperados`
**Descrição**: Lista de cooperados em operações de cooperativas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | FK |
| `nu_ordem` | INTEGER | FK |
| `cpf_cnpj` | TEXT | CPF/CNPJ do cooperado |
| `tipo_pessoa` | CHAR(1) | 'F' ou 'J' |
| `valor_parcela` | NUMERIC | Valor da parcela do cooperado |
| `cd_programa` | CHAR(4) | FK → programa |

---

#### `sicor_saldos`
**Descrição**: Saldos das operações (série temporal).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | INTEGER | FK |
| `nu_ordem` | INTEGER | FK |
| `ano_base` | INTEGER | Ano de referência |
| `mes_base` | INTEGER | Mês de referência |
| `cd_situacao_operacao` | INTEGER | FK → situacaooperacao |
| `vl_ultimo_dia` | NUMERIC | Saldo no último dia do mês |
| `vl_medio_diario` | NUMERIC | Saldo médio diário |
| `vl_medio_diario_vincendo` | NUMERIC | Saldo médio vincendo |

---

#### `sicor_excecoes`
**Descrição**: Exceções de operações (dispensas IBAMA, UC, etc.).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `ref_bacen` | BIGINT | FK |
| `ref_bacen_efetivo` | BIGINT | Ref_bacen não mascarado |
| `ordem` | BIGINT | Ordem |
| `def_ib_ind_dispensa_ibama` | BIGINT | Indicador de dispensa IBAMA |
| `def_ib_ind_uc` | BIGINT | Indicador de UC |
| `def_ib_ind_floresta_publ` | BIGINT | Indicador de floresta pública |

---

#### `sicor_glebas_mpoly`
**Descrição**: Glebas agregadas por operação (MultiPolygon).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | BIGINT | **PK** |
| `ref_bacen` | INTEGER | FK |
| `nu_ordem` | INTEGER | FK |
| `data_emissao_contrato` | DATE | Data de emissão |
| `geom` | GEOMETRY | Geometria MultiPolygon |

---

### 3.11 TABELAS AUXILIARES DE GLEBAS

#### `glebas_area_estudo`
**Descrição**: Glebas filtradas para área de estudo específica.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | BIGINT | **PK** |
| `ref_bacen` | INTEGER | FK |
| `nu_ordem` | INTEGER | FK |
| `nu_indice` | INTEGER | Índice da gleba |
| `data_emissao_contrato` | DATE | Data de emissão |
| `geom` | GEOMETRY | Geometria |
| `area_gleba` | NUMERIC | Área calculada |
| `perimetro_gleba` | NUMERIC | Perímetro |
| `area_menor_retangulo_envolvente` | NUMERIC | Área MBR |
| `area_menor_circulo_envolvente` | NUMERIC | Área círculo envolvente |

---

#### `glebas_fora_brasil`
**Descrição**: Glebas com coordenadas fora do território brasileiro (erros).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | BIGINT | **PK** |
| `ref_bacen` | INTEGER | FK |
| `nu_ordem` | INTEGER | FK |
| `nu_indice` | INTEGER | Índice da gleba |
| `data_emissao_contrato` | DATE | Data de emissão |
| `geom` | GEOMETRY | Geometria (coordenadas inválidas) |

---

### 3.12 TABELAS CAR COMPLETAS (Por Estado)

As tabelas do CAR seguem padrão de nomenclatura: `car_[tema]_[uf]`

#### Estrutura Padrão das Tabelas CAR:

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` / `gid` | INTEGER | **PK** |
| `cod_imovel` | VARCHAR | Código do imóvel CAR |
| `cod_tema` | VARCHAR | Código do tema |
| `nom_tema` | VARCHAR | Nome do tema |
| `num_area` | NUMERIC | Área em hectares |
| `ind_status` | VARCHAR | Status do registro |
| `des_condic` | VARCHAR | Descrição da condição |
| `geom` | GEOMETRY | Geometria |

#### Tabelas CAR Disponíveis:

**Área de Preservação Permanente (APP):**
- `car_app` (genérica)
- `car_app_pr`, `car_app_rs`, `car_app_sc`

**Área Consolidada:**
- `car_area_consolidada_pr`, `car_area_consolidada_rs`, `car_area_consolidada_sc`

**Área de Pousio:**
- `car_area_pousio_pr`, `car_area_pousio_rs`, `car_area_pousio_sc`

**Hidrografia:**
- `car_hidrografia_pr`, `car_hidrografia_rs`, `car_hidrografia_sc`

**Reserva Legal:**
- `car_reserva_legal_pr`, `car_reserva_legal_rs`, `car_reserva_legal_sc`

**Servidão Administrativa:**
- `car_servidao_administrativa_pr`, `car_servidao_administrativa_rs`, `car_servidao_administrativa_sc`

**Uso Restrito:**
- `car_uso_restrito_pr`, `car_uso_restrito_rs`, `car_uso_restrito_sc`

**Vegetação Nativa:**
- `car_vegetacao_nativa_pr`, `car_vegetacao_nativa_rs`, `car_vegetacao_nativa_sc`

**Área do Imóvel:**
- `car_area_imovel` - Área total do imóvel CAR
- `car_area_imovel_invalida` - Imóveis com geometria inválida

---

### 3.13 TABELAS FUNDIÁRIAS ADICIONAIS

#### `sigef`
**Descrição**: Sistema de Gestão Fundiária (INCRA) - parcelas certificadas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `codigo_imo` | VARCHAR | Código do imóvel |
| `nome_area` | VARCHAR | Nome da área |
| `status` | VARCHAR | Status da certificação |
| `parcela_co` | VARCHAR | Código da parcela |
| `situacao_i` | VARCHAR | Situação do imóvel |
| `art` | VARCHAR | ART do responsável técnico |
| `rt` | VARCHAR | Responsável técnico |
| `data_submi` | DATE | Data de submissão |
| `data_aprov` | DATE | Data de aprovação |
| `registro_d` | DATE | Data de registro |
| `municipio_` | INTEGER | Código do município |
| `uf_id` | INTEGER | ID da UF |
| `geom` | GEOMETRY | Geometria |

---

#### `imovel_certif_snci`
**Descrição**: Imóveis certificados pelo SNCI (Sistema Nacional de Certificação de Imóveis).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `cod_imovel` | VARCHAR | Código do imóvel |
| `nome_imove` | VARCHAR | Nome do imóvel |
| `num_certif` | VARCHAR | Número do certificado |
| `data_certi` | DATE | Data da certificação |
| `num_proces` | VARCHAR | Número do processo |
| `qtd_area_p` | VARCHAR | Área do imóvel |
| `cod_profis` | VARCHAR | Código do profissional |
| `sr` | VARCHAR | Superintendência Regional |
| `uf_municip` | VARCHAR | UF/Município |
| `geom` | GEOMETRY | Geometria |

---

#### `cnfp_2022`
**Descrição**: Cadastro Nacional de Florestas Públicas.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `nome` | VARCHAR | Nome da floresta |
| `codigo` | VARCHAR | Código |
| `categoria` | VARCHAR | Categoria |
| `classe` | VARCHAR | Classe |
| `tipo` | VARCHAR | Tipo |
| `orgao` | VARCHAR | Órgão gestor |
| `governo` | VARCHAR | Esfera de governo |
| `protecao` | VARCHAR | Tipo de proteção |
| `uf` | VARCHAR | UF |
| `bioma` | VARCHAR | Bioma |
| `area_ha` | NUMERIC | Área em hectares |
| `anocriacao` | DOUBLE | Ano de criação |
| `atolegal` | VARCHAR | Ato legal |
| `geom` | GEOMETRY | Geometria |

---

#### `cadastro_empregadores`
**Descrição**: Lista de empregadores com trabalho análogo à escravidão ("Lista Suja").

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `id` | INTEGER | **PK** |
| `empregador` | TEXT | Nome do empregador |
| `cnpj_cpf` | TEXT | CNPJ ou CPF |
| `estabelecimento` | TEXT | Nome do estabelecimento |
| `uf` | CHAR(2) | UF |
| `cnae` | TEXT | Código CNAE |
| `ano_acao_fiscal` | INTEGER | Ano da ação fiscal |
| `numero_trabalhadores` | INTEGER | Número de trabalhadores resgatados |
| `data_decisao_adm` | DATE | Data da decisão administrativa |
| `data_inclusao_cad` | TEXT | Data de inclusão no cadastro |

---

#### `embargo_ibama_20240508`
**Descrição**: Embargos IBAMA (versão maio/2024) - estrutura mais detalhada.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `num_auto_i` | VARCHAR | Número do auto de infração |
| `ser_auto_i` | VARCHAR | Série do auto |
| `dat_embarg` | DATE | Data do embargo |
| `des_infrac` | VARCHAR | Descrição da infração |
| `nom_pessoa` | VARCHAR | Nome da pessoa autuada |
| `cpf_cnpj_i` | VARCHAR | CPF/CNPJ |
| `nom_propri` | VARCHAR | Nome da propriedade |
| `nom_munici` | VARCHAR | Município |
| `sig_uf` | VARCHAR | UF |
| `qtd_area_e` | VARCHAR | Área embargada |
| `sit_desemb` | VARCHAR | Situação do desembargo |
| `dat_desemb` | DATE | Data do desembargo |
| `geom` | GEOMETRY | Geometria |
| ... | ... | (muitas outras colunas de detalhe) |

---

### 3.14 TABELAS DE ÁREA DE ESTUDO

#### `area_estudo`
**Descrição**: Delimitação da área de estudo do projeto (mesorregiões).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `gid` | INTEGER | **PK** |
| `id` | DOUBLE | ID |
| `cd_meso` | VARCHAR | Código da mesorregião |
| `nm_meso` | VARCHAR | Nome da mesorregião |
| `sigla_uf` | VARCHAR | UF |
| `area_km2` | NUMERIC | Área em km² |
| `geom` | GEOMETRY | Geometria |

---

### 3.15 TABELAS DE SISTEMA PostGIS

#### `geometry_columns`
**Descrição**: Catálogo de colunas geométricas (view do PostGIS).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `f_table_catalog` | VARCHAR | Catálogo |
| `f_table_schema` | NAME | Schema |
| `f_table_name` | NAME | Nome da tabela |
| `f_geometry_column` | NAME | Nome da coluna geométrica |
| `coord_dimension` | INTEGER | Dimensão (2D, 3D) |
| `srid` | INTEGER | SRID |
| `type` | VARCHAR | Tipo geométrico |

---

#### `geography_columns`
**Descrição**: Catálogo de colunas geográficas (view do PostGIS).

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `f_table_catalog` | NAME | Catálogo |
| `f_table_schema` | NAME | Schema |
| `f_table_name` | NAME | Nome da tabela |
| `f_geography_column` | NAME | Nome da coluna |
| `coord_dimension` | INTEGER | Dimensão |
| `srid` | INTEGER | SRID |
| `type` | TEXT | Tipo |

---

#### `spatial_ref_sys`
**Descrição**: Sistemas de Referência Espacial disponíveis.

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `srid` | INTEGER | **PK** - Código SRID |
| `auth_name` | VARCHAR | Autoridade (EPSG, etc.) |
| `auth_srid` | INTEGER | SRID da autoridade |
| `srtext` | VARCHAR | Definição WKT |
| `proj4text` | VARCHAR | Definição Proj4 |

> **Uso comum**: `SELECT * FROM spatial_ref_sys WHERE srid = 4674;` (SIRGAS 2000)

---

## 4. RELACIONAMENTOS ENTRE TABELAS

### 4.1 Diagrama de Relacionamentos Principal

```
sicor_operacao_basica_estado (ref_bacen, nu_ordem)
    │
    ├──FK→ empreendimento (cd_empreendimento)
    ├──FK→ programa (cd_programa)
    │       └──1:N→ subprograma (codigo_programa)
    ├──FK→ fonterecursos (cd_fonte_recurso)
    ├──FK→ categoriaemitente (cd_categ_emitente)
    ├──FK→ tipogarantiaempreendimento (cd_tipo_seguro)
    ├──FK→ ifssicor (cnpj_if)
    ├──FK→ instrumentocredito (cd_inst_credito)
    ├──FK→ ciclocultivarproagro (cd_ciclo_cultivar)
    ├──FK→ tiposoloproagro (cd_tipo_solo)
    ├──FK→ tipoagropecuaria (cd_tipo_agricultura)
    ├──FK→ tipoirrigacao (cd_tipo_irrigacao)
    ├──FK→ tipocultivo (cd_tipo_cultivo)
    ├──FK→ tipointegracao (cd_tipo_intgr_consor)
    ├──FK→ graosemente (cd_tipo_grao_semente)
    ├──FK→ fasecicloproducao (cd_fase_ciclo_producao)
    ├──FK→ encargosfinanceiroscomplementares (cd_tipo_encarg_financ)
    │
    ├──1:N→ sicor_glebas_wkt (ref_bacen, nu_ordem)
    ├──1:N→ sicor_glebas (ref_bacen, nu_ordem)
    ├──1:N→ sicor_glebas_mpoly (ref_bacen, nu_ordem) [agregado]
    │
    ├──1:N→ sicor_cop_basico (ref_bacen, nu_ordem, cd_evento)
    │           ├──FK→ eventoproagro (cd_evento)
    │           ├──FK→ statuscopproagro (cd_status)
    │           └──1:1→ sicor_complemento_cop (ref_bacen, nu_ordem, cd_evento)
    │
    ├──1:N→ sicor_rcp_basico (ref_bacen, nu_ordem)
    │           ├──1:N→ sicor_rcp_glebas (ref_bacen, nu_ordem)
    │           ├──1:N→ sicor_rcp_glebas_wkt (ref_bacen, nu_ordem)
    │           └──1:1→ sicor_complemento_rcp (ref_bacen, nu_ordem)
    │
    ├──1:N→ sicor_parcelas_proagro (ref_bacen, nu_ordem)
    │           ├──FK→ naturezaproagro (cd_natureza_parcela)
    │           ├──FK→ statusparcelaproagro (cd_status)
    │           └──FK→ instanciaproagro (cd_instancia)
    │
    ├──1:N→ sicor_sumula_julgamento_basico (ref_bacen, nu_ordem)
    │           └──FK→ instanciaproagro (cd_instancia)
    │
    ├──1:1→ sicor_complemento_operacao_basica (ref_bacen, nu_ordem)
    │           └── cd_ibge_municipio → municipios_2022 (cd_mun)
    │           └── cd_ibge_municipio → zarc (geocodigo)
    │
    ├──1:N→ sicor_mutuarios (ref_bacen)
    │           └──FK→ tipobeneficiario (cd_tipo_beneficiario)
    │
    ├──1:N→ sicor_propriedades (ref_bacen, nu_ordem)
    │           └── cd_car → car_area_imovel (cod_imovel_sicor)
    │
    ├──1:N→ sicor_desclassificacao (ref_bacen, nu_ordem)
    │           └──FK→ motivodesclassificacao (cd_motivo_desc)
    │
    ├──1:N→ sicor_liberacao_recursos (ref_bacen, nu_ordem)
    │
    ├──1:N→ sicor_parcelas_desembolso (ref_bacen, nu_ordem)
    │
    ├──1:N→ sicor_lista_cooperados (ref_bacen, nu_ordem)
    │
    ├──1:N→ sicor_saldos (ref_bacen, nu_ordem)
    │           └──FK→ situacaooperacao (cd_situacao_operacao)
    │
    └──1:1→ sicor_excecoes (ref_bacen)
```

### 4.2 Relacionamento ZARC

```
sicor_cop_basico
    │
    ├── cd_tipo_solo ─────────────────→ zarc.cod_solo
    ├── cd_ciclo_cultivar → ciclocultivarproagro.descricao_ciclo → zarc.grupo
    │
    └── sicor_complemento_operacao_basica.cd_ibge_municipio → zarc.geocodigo

Validação de plantio:
  cop.dt_inicio_plantio <= zarc.data_final_decendio 
  AND zarc.data_inicial_decendio <= cop.dt_fim_plantio
  AND zarc.risco = 0 (plantio dentro da janela recomendada)
```

---

## 5. PADRÕES DE CONSULTA

### 5.1 JOIN Básico: Operação + Empreendimento

```sql
SELECT op.ref_bacen, op.nu_ordem, op.dt_emissao, op.vl_parc_credito,
       emp.produto, emp.finalidade, emp.modalidade, emp.atividade
  FROM sicor_operacao_basica_estado AS op
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 WHERE emp.finalidade = 'custeio'
   AND emp.produto = 'soja';
```

### 5.2 JOIN com Município (via Complemento)

```sql
SELECT op.ref_bacen, op.nu_ordem, 
       mun.nm_mun, mun.sigla_uf
  FROM sicor_operacao_basica_estado AS op
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
 INNER JOIN municipios_2022 AS mun
    ON comp.cd_ibge_municipio = mun.cd_mun;
```

### 5.3 Pedidos Proagro com Evento

```sql
SELECT cop.ref_bacen, cop.nu_ordem, cop.dt_comunicacao,
       evento.nome_evento,
       status.descricao AS status_cop
  FROM sicor_cop_basico AS cop
 INNER JOIN eventoproagro AS evento
    ON cop.cd_evento = evento.cd_evento
 INNER JOIN statuscopproagro AS status
    ON cop.cd_status = status.cd_status;
```

### 5.4 Consulta Completa: COP + Operação + Empreendimento

```sql
SELECT cop.ref_bacen, cop.nu_ordem, cop.dt_comunicacao,
       evento.nome_evento,
       emp.produto, emp.modalidade,
       op.vl_parc_credito, op.vl_area_financ,
       op.dt_emissao, op.cd_estado
  FROM sicor_cop_basico AS cop
 INNER JOIN eventoproagro AS evento
    ON cop.cd_evento = evento.cd_evento
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 WHERE extract(YEAR FROM op.dt_emissao) = 2023
   AND emp.finalidade = 'custeio'
   AND evento.nome_evento = 'Seca';
```

### 5.5 Validação ZARC

```sql
SELECT cop.ref_bacen, cop.nu_ordem,
       cop.dt_inicio_plantio, cop.dt_fim_plantio,
       zarc.decendio, zarc.risco,
       zarc.data_inicial_decendio, zarc.data_final_decendio,
       CASE WHEN zarc.risco = 0 THEN 'DENTRO DA JANELA' 
            ELSE 'FORA DA JANELA' END AS validacao_zarc
  FROM sicor_cop_basico AS cop
 INNER JOIN ciclocultivarproagro AS ciclo
    ON cop.cd_ciclo_cultivar = ciclo.cd_ciclo_cultivar
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
  LEFT JOIN zarc
    ON comp.cd_ibge_municipio = zarc.geocodigo
   AND cop.cd_tipo_solo = zarc.cod_solo
   AND ciclo.descricao_ciclo = zarc.grupo
   AND zarc.cultura = emp.produto
   AND cop.dt_inicio_plantio <= zarc.data_final_decendio 
   AND zarc.data_inicial_decendio <= cop.dt_fim_plantio
 WHERE emp.produto = 'soja';
```

### 5.6 Agregação por Ano/Estado

```sql
SELECT extract(YEAR FROM dt_emissao) AS ano,
       cd_estado,
       COUNT(*) AS num_contratos,
       SUM(vl_parc_credito) AS valor_total,
       AVG(vl_area_financ) AS area_media
  FROM sicor_operacao_basica_estado
 WHERE cd_tipo_seguro IN ('1', '2')  -- Proagro
 GROUP BY extract(YEAR FROM dt_emissao), cd_estado
 ORDER BY ano DESC, valor_total DESC;
```

### 5.7 Valores Pagos Proagro

```sql
SELECT op.cd_estado,
       evento.nome_evento,
       COUNT(DISTINCT cop.ref_bacen) AS num_pedidos,
       SUM(parcela.vl_pago) AS total_pago
  FROM sicor_cop_basico AS cop
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN eventoproagro AS evento
    ON cop.cd_evento = evento.cd_evento
  LEFT JOIN sicor_parcelas_proagro AS parcela
    ON cop.ref_bacen = parcela.ref_bacen AND cop.nu_ordem = parcela.nu_ordem
 WHERE parcela.vl_pago > 0
 GROUP BY op.cd_estado, evento.nome_evento
 ORDER BY total_pago DESC;
```

---

## 6. FUNÇÕES POSTGIS

### 6.1 Conversão WKT para Geometry

```sql
-- Converter WKT para geometry com SRID 4674
SELECT (ST_Force2D(gt_geometria::geometry))::geometry(geometry, 4674) AS geom
  FROM sicor_glebas_wkt;
```

> **IMPORTANTE**: Glebas do SICOR podem ter coordenadas 3D. Usar `ST_Force2D()` para converter para 2D.

### 6.2 Cálculo de Área e Perímetro

```sql
-- Área em metros quadrados (projeção para calcular)
SELECT ST_Area(ST_Transform(geom, 100000)) AS area_m2,
       ST_Area(ST_Transform(geom, 100000)) / 10000 AS area_ha
  FROM sicor_glebas;

-- Perímetro em metros
SELECT ST_Perimeter(ST_Transform(geom, 100000)) AS perimetro_m
  FROM sicor_glebas;
```

### 6.3 Relacionamentos Espaciais

```sql
-- Verificar se gleba está dentro de município
SELECT g.ref_bacen, m.nm_mun
  FROM sicor_glebas AS g
 INNER JOIN municipios_2022 AS m
    ON ST_Within(g.geom, m.geom);

-- Verificar sobreposição com área embargada
SELECT g.ref_bacen, e.num_auto_i, e.data_embargo
  FROM sicor_glebas AS g
 INNER JOIN embargo_ibama_20241106 AS e
    ON ST_Intersects(g.geom, e.geom);

-- Área de sobreposição
SELECT g.ref_bacen,
       ST_Area(ST_Transform(ST_Intersection(g.geom, e.geom), 100000)) AS area_sobreposta_m2
  FROM sicor_glebas AS g
 INNER JOIN embargo_ibama_20241106 AS e
    ON ST_Intersects(g.geom, e.geom);
```

### 6.4 Validação de Geometrias

```sql
-- Verificar geometrias inválidas
SELECT ref_bacen, nu_ordem, nu_indice,
       ST_IsValidReason(geom) AS motivo_invalidade
  FROM sicor_glebas
 WHERE NOT ST_IsValid(geom);

-- Corrigir geometrias inválidas
SELECT ref_bacen, ST_MakeValid(geom) AS geom_corrigida
  FROM sicor_glebas
 WHERE NOT ST_IsValid(geom);
```

### 6.5 Centroide e Bounding Box

```sql
-- Centroide da gleba
SELECT ref_bacen, 
       ST_X(ST_Centroid(geom)) AS longitude,
       ST_Y(ST_Centroid(geom)) AS latitude
  FROM sicor_glebas;

-- Bounding box
SELECT ref_bacen,
       ST_XMin(geom) AS lon_min, ST_XMax(geom) AS lon_max,
       ST_YMin(geom) AS lat_min, ST_YMax(geom) AS lat_max
  FROM sicor_glebas;
```

### 6.6 Buffer e Proximidade

```sql
-- Glebas a menos de 1km de área embargada
SELECT DISTINCT g.ref_bacen
  FROM sicor_glebas AS g
 INNER JOIN embargo_ibama_20241106 AS e
    ON ST_DWithin(
         ST_Transform(g.geom, 100000), 
         ST_Transform(e.geom, 100000), 
         1000  -- 1000 metros
       );
```

### 6.7 Exportar GeoJSON

```sql
-- Exportar geometria como GeoJSON
SELECT ref_bacen, ST_AsGeoJSON(geom) AS geojson
  FROM sicor_glebas
 LIMIT 10;
```

---

## 7. EXEMPLOS PRÁTICOS

### 7.1 Pedidos de Proagro por Seca no RS em 2023

```sql
SELECT cop.ref_bacen, cop.nu_ordem, cop.dt_comunicacao,
       emp.produto,
       op.vl_parc_credito,
       comp.cd_ibge_municipio,
       mun.nm_mun
  FROM sicor_cop_basico AS cop
 INNER JOIN eventoproagro AS evento
    ON cop.cd_evento = evento.cd_evento
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
  LEFT JOIN municipios_2022 AS mun
    ON comp.cd_ibge_municipio = mun.cd_mun
 WHERE evento.nome_evento = 'Seca'
   AND op.cd_estado = 'RS'
   AND extract(YEAR FROM cop.dt_comunicacao) = 2023
 ORDER BY cop.dt_comunicacao;
```

### 7.2 Top 10 Municípios com Maior Valor Pago Proagro

```sql
SELECT mun.nm_mun, mun.sigla_uf,
       COUNT(DISTINCT parcela.ref_bacen) AS num_pedidos,
       SUM(parcela.vl_pago) AS total_pago
  FROM sicor_parcelas_proagro AS parcela
 INNER JOIN sicor_operacao_basica_estado AS op
    ON parcela.ref_bacen = op.ref_bacen AND parcela.nu_ordem = op.nu_ordem
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
 INNER JOIN municipios_2022 AS mun
    ON comp.cd_ibge_municipio = mun.cd_mun
 WHERE parcela.vl_pago > 0
 GROUP BY mun.nm_mun, mun.sigla_uf
 ORDER BY total_pago DESC
 LIMIT 10;
```

### 7.3 Glebas em Áreas Embargadas (Análise de Risco)

```sql
WITH glebas_embargadas AS (
    SELECT g.ref_bacen, g.nu_ordem, g.nu_indice,
           e.num_auto_i, e.data_embargo, e.des_infrac,
           ST_Area(ST_Transform(ST_Intersection(g.geom, e.geom), 100000)) / 10000 AS area_sobreposta_ha
      FROM sicor_glebas AS g
     INNER JOIN embargo_ibama_20241106 AS e
        ON ST_Intersects(g.geom, e.geom)
     WHERE g.data_emissao_contrato >= '2020-01-01'
)
SELECT ge.ref_bacen, ge.nu_ordem,
       op.dt_emissao, emp.produto,
       ge.num_auto_i, ge.data_embargo,
       ge.area_sobreposta_ha
  FROM glebas_embargadas AS ge
 INNER JOIN sicor_operacao_basica_estado AS op
    ON ge.ref_bacen = op.ref_bacen AND ge.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 WHERE ge.data_embargo < op.dt_emissao  -- Embargo anterior ao contrato
 ORDER BY ge.area_sobreposta_ha DESC;
```

### 7.4 Distribuição de Eventos por Cultura

```sql
SELECT emp.produto,
       evento.nome_evento,
       COUNT(*) AS num_pedidos,
       ROUND(COUNT(*)::numeric / SUM(COUNT(*)) OVER (PARTITION BY emp.produto) * 100, 2) AS percentual
  FROM sicor_cop_basico AS cop
 INNER JOIN eventoproagro AS evento
    ON cop.cd_evento = evento.cd_evento
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 WHERE emp.finalidade = 'custeio'
   AND emp.modalidade = 'lavoura'
   AND emp.produto IN ('soja', 'milho', 'trigo', 'feijão')
 GROUP BY emp.produto, evento.nome_evento
 ORDER BY emp.produto, num_pedidos DESC;
```

### 7.5 Análise Temporal de Pedidos Proagro

```sql
WITH pedidos_mensais AS (
    SELECT date_trunc('month', cop.dt_comunicacao) AS mes,
           evento.nome_evento,
           COUNT(*) AS num_pedidos
      FROM sicor_cop_basico AS cop
     INNER JOIN eventoproagro AS evento
        ON cop.cd_evento = evento.cd_evento
     WHERE cop.dt_comunicacao >= '2020-01-01'
     GROUP BY date_trunc('month', cop.dt_comunicacao), evento.nome_evento
)
SELECT mes, nome_evento, num_pedidos,
       SUM(num_pedidos) OVER (PARTITION BY nome_evento ORDER BY mes) AS acumulado
  FROM pedidos_mensais
 ORDER BY mes, nome_evento;
```

---

## 8. ARMADILHAS COMUNS

### 8.1 Geometrias 3D

**PROBLEMA**: Glebas do SICOR podem ter coordenadas Z, causando erro em funções 2D.

**SOLUÇÃO**: Sempre usar `ST_Force2D()` ao converter de WKT:
```sql
SELECT (ST_Force2D(gt_geometria::geometry))::geometry(geometry, 4674) AS geom
  FROM sicor_glebas_wkt;
```

### 8.2 Geometrias Inválidas

**PROBLEMA**: Algumas glebas têm geometrias topologicamente inválidas.

**SOLUÇÃO**: Filtrar ou corrigir:
```sql
-- Filtrar válidas
WHERE ST_IsValid(geom)

-- Corrigir
SELECT ST_MakeValid(geom) AS geom_corrigida
```

### 8.3 Glebas Fora do Brasil

**PROBLEMA**: Algumas glebas têm coordenadas erradas (fora do território brasileiro).

**SOLUÇÃO**: Filtrar por bounding box do Brasil:
```sql
WHERE ST_X(ST_Centroid(geom)) BETWEEN -74 AND -34
  AND ST_Y(ST_Centroid(geom)) BETWEEN -34 AND 6
```

### 8.4 Múltiplos Eventos por Pedido

**PROBLEMA**: Um contrato pode ter múltiplos eventos na mesma COP.

**ATENÇÃO**: A chave da `sicor_cop_basico` inclui `cd_evento`, então podem existir múltiplas linhas para o mesmo `(ref_bacen, nu_ordem)`:
```sql
-- Verificar contratos com múltiplos eventos
SELECT ref_bacen, nu_ordem, COUNT(*) AS num_eventos
  FROM sicor_cop_basico
 GROUP BY ref_bacen, nu_ordem
HAVING COUNT(*) > 1;
```

### 8.5 JOIN com ZARC

**PROBLEMA**: O ZARC tem múltiplas linhas por município/cultura (uma para cada decêndio).

**SOLUÇÃO**: Filtrar pelo decêndio que corresponde à data de plantio:
```sql
WHERE cop.dt_inicio_plantio <= zarc.data_final_decendio 
  AND zarc.data_inicial_decendio <= cop.dt_fim_plantio
```

### 8.6 Tipo de Seguro vs Proagro

**PROBLEMA**: Nem todas as operações têm Proagro.

**SOLUÇÃO**: Filtrar por `cd_tipo_seguro`:
```sql
WHERE cd_tipo_seguro IN ('1', '2')  -- '1' = Proagro, '2' = Proagro Mais
```

### 8.7 Cálculo de Área em Coordenadas Geográficas

**PROBLEMA**: `ST_Area()` em EPSG:4674 retorna graus², não metros².

**SOLUÇÃO**: Transformar para projeção métrica:
```sql
SELECT ST_Area(ST_Transform(geom, 100000)) AS area_m2  -- Projeção métrica genérica
```

### 8.8 Performance em Consultas Espaciais

**PROBLEMA**: Consultas espaciais podem ser lentas sem índice.

**SOLUÇÃO**: Garantir índice GiST:
```sql
CREATE INDEX sicor_glebas_geom_idx ON sicor_glebas USING GiST(geom);
```

---

## 9. TEMPLATES DE CONSULTA

### 9.1 Template: Busca de Operações

```sql
-- [SUBSTITUIR]: ano, estado, produto, finalidade
SELECT op.ref_bacen, op.nu_ordem, op.dt_emissao, op.vl_parc_credito,
       emp.produto, emp.finalidade, emp.modalidade,
       if_sicor.nome_if AS instituicao_financeira,
       categ.descricao AS categoria_produtor
  FROM sicor_operacao_basica_estado AS op
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
  LEFT JOIN ifssicor AS if_sicor
    ON op.cnpj_if = if_sicor.cnpj_if
  LEFT JOIN categoriaemitente AS categ
    ON op.cd_categ_emitente = categ.cd_categ_emitente
 WHERE extract(YEAR FROM op.dt_emissao) = {ANO}
   AND op.cd_estado = '{ESTADO}'
   AND emp.produto = '{PRODUTO}'
   AND emp.finalidade = '{FINALIDADE}'
 ORDER BY op.dt_emissao
 LIMIT 100;
```

### 9.2 Template: Pedidos Proagro

```sql
-- [SUBSTITUIR]: ano, evento, estado
SELECT cop.ref_bacen, cop.nu_ordem, cop.dt_comunicacao,
       evento.nome_evento,
       emp.produto,
       op.vl_parc_credito,
       op.vl_area_financ,
       mun.nm_mun, mun.sigla_uf
  FROM sicor_cop_basico AS cop
 INNER JOIN eventoproagro AS evento
    ON cop.cd_evento = evento.cd_evento
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
  LEFT JOIN municipios_2022 AS mun
    ON comp.cd_ibge_municipio = mun.cd_mun
 WHERE extract(YEAR FROM cop.dt_comunicacao) = {ANO}
   AND evento.nome_evento = '{EVENTO}'
   AND op.cd_estado = '{ESTADO}'
 ORDER BY cop.dt_comunicacao;
```

### 9.3 Template: Validação ZARC

```sql
-- [SUBSTITUIR]: ano, produto
SELECT cop.ref_bacen, cop.nu_ordem,
       emp.produto,
       cop.dt_inicio_plantio, cop.dt_fim_plantio,
       zarc.decendio, zarc.risco,
       zarc.data_inicial_decendio, zarc.data_final_decendio,
       mun.nm_mun,
       CASE 
           WHEN zarc.risco = 0 THEN 'DENTRO DA JANELA ZARC'
           WHEN zarc.risco = 1 THEN 'FORA DA JANELA ZARC'
           ELSE 'SEM CORRESPONDÊNCIA ZARC'
       END AS validacao
  FROM sicor_cop_basico AS cop
 INNER JOIN ciclocultivarproagro AS ciclo
    ON cop.cd_ciclo_cultivar = ciclo.cd_ciclo_cultivar
 INNER JOIN sicor_operacao_basica_estado AS op
    ON cop.ref_bacen = op.ref_bacen AND cop.nu_ordem = op.nu_ordem
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
  LEFT JOIN municipios_2022 AS mun
    ON comp.cd_ibge_municipio = mun.cd_mun
  LEFT JOIN zarc
    ON comp.cd_ibge_municipio = zarc.geocodigo
   AND cop.cd_tipo_solo = zarc.cod_solo
   AND ciclo.descricao_ciclo = zarc.grupo
   AND LOWER(zarc.cultura) = LOWER(emp.produto)
   AND cop.dt_inicio_plantio <= zarc.data_final_decendio 
   AND zarc.data_inicial_decendio <= cop.dt_fim_plantio
 WHERE extract(YEAR FROM op.dt_emissao) = {ANO}
   AND emp.produto = '{PRODUTO}'
 ORDER BY cop.ref_bacen;
```

### 9.4 Template: Análise Espacial de Glebas

```sql
-- [SUBSTITUIR]: ano, área_minima_ha
SELECT g.ref_bacen, g.nu_ordem, g.nu_indice,
       ST_Area(ST_Transform(g.geom, 100000)) / 10000 AS area_ha,
       ST_X(ST_Centroid(g.geom)) AS longitude,
       ST_Y(ST_Centroid(g.geom)) AS latitude,
       emp.produto,
       mun.nm_mun, mun.sigla_uf
  FROM sicor_glebas AS g
 INNER JOIN sicor_operacao_basica_estado AS op
    ON g.ref_bacen = op.ref_bacen AND g.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 INNER JOIN sicor_complemento_operacao_basica AS comp
    ON op.ref_bacen = comp.ref_bacen AND op.nu_ordem = comp.nu_ordem
  LEFT JOIN municipios_2022 AS mun
    ON comp.cd_ibge_municipio = mun.cd_mun
 WHERE extract(YEAR FROM g.data_emissao_contrato) = {ANO}
   AND ST_IsValid(g.geom)
   AND ST_Area(ST_Transform(g.geom, 100000)) / 10000 >= {AREA_MINIMA_HA}
 ORDER BY area_ha DESC
 LIMIT 100;
```

### 9.5 Template: Sobreposição com Áreas Restritas

```sql
-- [SUBSTITUIR]: ano, tipo_restricao (embargo, TI, quilombo, UC)
WITH restricoes AS (
    -- Embargos IBAMA
    SELECT 'Embargo IBAMA' AS tipo, geom FROM embargo_ibama_20241106
    UNION ALL
    -- Terras Indígenas
    SELECT 'Terra Indígena' AS tipo, geom FROM terras_indigenas
    UNION ALL
    -- Quilombolas
    SELECT 'Quilombola' AS tipo, geom FROM areas_quilombolas
    UNION ALL
    -- Unidades de Conservação
    SELECT 'Unidade de Conservação' AS tipo, geom FROM cnuc_2024_02
)
SELECT g.ref_bacen, g.nu_ordem,
       r.tipo AS tipo_restricao,
       ST_Area(ST_Transform(ST_Intersection(g.geom, r.geom), 100000)) / 10000 AS area_sobreposta_ha,
       emp.produto,
       op.dt_emissao
  FROM sicor_glebas AS g
 INNER JOIN restricoes AS r
    ON ST_Intersects(g.geom, r.geom)
 INNER JOIN sicor_operacao_basica_estado AS op
    ON g.ref_bacen = op.ref_bacen AND g.nu_ordem = op.nu_ordem
 INNER JOIN empreendimento AS emp
    ON op.cd_empreendimento = emp.cd_empreendimento
 WHERE extract(YEAR FROM g.data_emissao_contrato) = {ANO}
   AND ST_IsValid(g.geom)
 ORDER BY area_sobreposta_ha DESC;
```

---

## 📝 INSTRUÇÕES PARA A LLM

Ao receber uma pergunta em linguagem natural sobre o banco SICOR/Proagro:

1. **Identifique o objetivo**: O que o usuário quer saber? (operações, pedidos Proagro, glebas, validação ZARC, sobreposição espacial, etc.)

2. **Identifique as tabelas necessárias**: Use o catálogo acima para determinar quais tabelas contêm os dados relevantes.

3. **Identifique os relacionamentos**: Use a seção de relacionamentos para construir os JOINs corretos.

4. **Aplique filtros**: Identifique critérios de filtro mencionados (ano, estado, cultura, evento, etc.)

5. **Use funções PostGIS quando necessário**: Para consultas espaciais, use as funções descritas na seção 6.

6. **Evite armadilhas**: Consulte a seção 8 para evitar erros comuns.

7. **Use os templates**: Adapte um template da seção 9 quando aplicável.

**SEMPRE**:
- Use aliases descritivos para tabelas
- Inclua comentários explicando a lógica quando relevante
- Limite resultados com `LIMIT` para consultas exploratórias
- Use `ST_IsValid(geom)` ao trabalhar com geometrias
- Use `ST_Force2D()` ao converter de WKT
- Lembre-se que `ref_bacen + nu_ordem` é a chave composta principal

---

*Knowledge Base versão 1.0 - Projeto Geotec Grupo 4*
*Última atualização: Janeiro/2026*
