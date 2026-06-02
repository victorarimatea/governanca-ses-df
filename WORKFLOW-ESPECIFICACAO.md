# Especificação do Workflow de Transcrição Documental

**Projeto:** Transcrição de Documento para Arquivamento em Markdown
**Responsável:** Diretor de Transformação Digital — SES-DF
**Versão:** 1.1 — 2026-06-01
**Status:** Ativo

---

## 1. Intenção do Comandante

### Missão

Transformar documentos regulatórios oficiais em arquivos Markdown estruturados, fiéis ao original, auditáveis, pesquisáveis e reutilizáveis — constituindo a camada documental primária do ecossistema `governanca-ses-df`.

### Estado Final Esperado

Cada documento transcrito deve satisfazer **todos** os critérios abaixo. Este é o benchmark de qualidade que o pipeline deve verificar autonomamente antes de considerar uma transcrição concluída:

#### 1.1 Estrutura obrigatória

- [ ] Front Matter YAML válido com todos os campos preenchidos (nenhum campo vazio ou "Não identificado" sem justificativa)
- [ ] Ficha Técnica Documental em tabela Markdown com 12 linhas
- [ ] Seção "Observações da Conversão" presente e não vazia
- [ ] Seção "Conteúdo Integral Transcrito" presente e com conteúdo substantivo
- [ ] Seção "Controle de Integridade" com status de conversão explícito

#### 1.2 Fidelidade ao original

- [ ] Nenhum elemento estrutural jurídico ausente (ver §2.1)
- [ ] Nenhum artefato de extração presente no conteúdo (ver §2.2)
- [ ] Assinatura(s) presente(s) e corretamente formatadas
- [ ] Texto integral sem truncamento — número de parágrafos coerente com o número de páginas
- [ ] Nenhum texto refluxado com quebras de linha espúrias (linhas de conteúdo com menos de 40 caracteres intercaladas com linhas longas são sinal de alerta)

#### 1.3 Organização visual

- [ ] Cada marcador estrutural jurídico inicia em seu próprio parágrafo (linha em branco antes)
- [ ] Tabelas Markdown válidas (header + separador + linhas de dados)
- [ ] Nenhum parágrafo inicia com letra minúscula (exceto continuação de texto, alíneas)

#### 1.4 Rastreabilidade

- [ ] `arquivo_original` corresponde ao nome exato do PDF fonte
- [ ] `fonte_externa` contém URL válida ou "Não identificado" justificado
- [ ] `versao_transcricao` reflete o número de revisões do documento

---

## 2. Definições Técnicas

### 2.1 Elementos Estruturais Jurídicos (devem iniciar novo parágrafo)

| Marcador | Exemplo | Regex de detecção |
|---|---|---|
| Artigo | `Art. 1º` | `^Art\.\s+\d+` |
| Parágrafo numerado | `§ 1º` | `^§\s*\d+` |
| Parágrafo único | `Parágrafo único.` | `^Parágrafo\s+[Úú]nico` |
| Inciso | `I -` `XIV —` | `^[IVX]+\s+[-–—]` |
| Alínea | `a)` `b)` | `^[a-z])\s` |
| Capítulo | `CAPÍTULO I` | `^CAPÍTULO\s` |
| Seção | `Seção I` | `^SEÇÃO\s` ou `^Seção\s` |
| Subseção | `SUBSEÇÃO I` | `^SUBSEÇÃO\s` |
| Título | `TÍTULO I` | `^TÍTULO\s` |
| Livro | `LIVRO I` | `^LIVRO\s` |
| Anexo | `ANEXO I` | `^ANEXO\s[IVX]+` |

### 2.2 Artefatos de Extração (devem ser filtrados)

| Tipo | Exemplo | Filtro aplicado |
|---|---|---|
| Timestamp DOU | `27/04/2026, 23:46` | Regex `^\d{2}/\d{2}/\d{4},\s+\d{2}:\d{2}` |
| URL de acesso | `https://www.in.gov.br/...` | Regex `^https?://` |
| Numeração de página | `3/118` | Regex `^\d+/\d+$` |
| Cabeçalho DOU | `DIÁRIO OFICIAL DA UNIÃO` | Exato |
| Metadados DOU | `Publicado em:` `Órgão:` `Edição:` `Seção:` `Página:` | Prefixo |
| Disclaimer DOU | `Este conteúdo não substitui o publicado na versão certificada.` | Exato |
| Cabeçalho SINJ-DF | `Lei XXXX de DD/MM/AAAA` | Regex `^Lei\s+\d+\s+de\s+\d{2}/\d{2}/\d{4}` |
| Rodapé BVS | `Saúde Legis - Sistema de Legislação da Saúde` | Regex `^Sa[ãú]de Legis` |
| Advertência BVS | `Este texto não substitui...` | Regex `^Este texto n` |

### 2.3 Tipos de Fonte e Método de Extração Recomendado

| Tipo de Fonte | Método | Observações na Conversão |
|---|---|---|
| Portal Planalto.gov.br | `extract_dict_mode` | "PDF extraído do portal Planalto.gov.br, processado localmente." |
| DOU/Imprensa Nacional (PDF) | `extract_blocks` + reflow | "PDF oficial gerado a partir do DOU/Imprensa Nacional, processado localmente." |
| SINJ-DF | `extract_blocks` + reflow | "PDF extraído do SINJ-DF, processado localmente." |
| BVS Saúde Legis | `extract_blocks` + reflow | "PDF extraído do BVS Saúde Legis, processado localmente." |
| Portais internacionais (WHO, IMDRF etc.) | `extract_blocks` + reflow | Indicar idioma original e portal específico |

---

## 3. Histórico de Problemas e Soluções

### P01 — Fragmentação de nomes e datas no bloco de assinaturas
**Sintoma:** Data e nomes dos signatários ficavam fragmentados ou fundidos com o sufixo de datação.
**Causa raiz:** `split_signature_names()` usava transição minúscula→maiúscula como critério de quebra.
**Solução:** Detectar sufixo `;` após data → manter linha intacta. Dividir nomes apenas em espaços duplos (`\s{2,}`).
**Status:** ✅ Resolvido.

### P02 — Artefatos DOU no conteúdo transcrito
**Sintoma:** URLs, timestamps e cabeçalhos DOU apareciam no corpo do documento.
**Causa raiz:** PDFs do DOU incluem metadados de acesso web no layout de página.
**Solução:** Lista de filtros (`SKIP_EXACT`, `SKIP_RE`) aplicados linha a linha.
**Status:** ✅ Resolvido.

### P03 — Cabeçalhos SINJ-DF e BVS repetidos entre páginas
**Sintoma:** "Lei 7215 de 02/01/2023" e "Saúde Legis..." apareciam como parágrafos de conteúdo.
**Causa raiz:** Cabeçalhos e rodapés de página tratados como texto normativo.
**Solução:** Regexes específicas adicionadas ao `SKIP_RE`.
**Status:** ✅ Resolvido.

### P04 — Marcadores estruturais jurídicos fundidos com parágrafos anteriores
**Sintoma:** `Art. 2º` aparecia como continuação do parágrafo do `Art. 1º`.
**Causa raiz:** O reflow juntava linhas sem verificar marcadores estruturais.
**Solução:** `LEGAL_STRUCT` regex aplicado durante o reflow e em `apply_legal_structure()`.
**Status:** ✅ Resolvido — implementado em v2.3 (dict-mode) e v1.2 (extract_blocks).

### P05 — Quebras de linha espúrias no texto corrido
**Sintoma:** Parágrafos com texto quebrado palavra por palavra.
**Causa raiz:** `extract_blocks` preservava todas as quebras internas dos blocos.
**Solução:** `reflow_block_lines()` — linhas juntadas com espaço exceto antes de marcadores jurídicos.
**Status:** ✅ Resolvido.

### P06 — ANEXO II da Portaria 3.233/2024 — cobertura parcial de municípios
**Sintoma:** Parser original recuperava apenas ~75% dos 5.570 municípios.
**Causa raiz:** Layout de tabela com coluna de valor extraída separadamente.
**Solução:** Parser regex com `MUN_PATTERN` sobre texto achatado + merge de blocos partidos + fallback `MUN_NOVAL`.
**Resultado:** 5.571 municípios; 4.241 com valor, 1.330 marcados como N/I.
**Status:** ✅ Resolvido com ressalva documentada.

### P07 — Redação imprecisa de "Observações da Conversão" para PDFs do DOU
**Sintoma:** Redação sugeria que o DOU foi acessado diretamente, não o PDF local.
**Causa raiz:** Redação genérica sem distinguir fonte de origem do artefato processado.
**Solução:** Funções `obs_dou_pdf()` e `obs_portal_pdf()` com redação padronizada.
**Status:** ✅ Resolvido.

### P08 — Arquivo original inutilizável (capa visual sem conteúdo textual)
**Sintoma:** PDF de 1 página sem texto extraível — identificado no EU AI Act Overview 2024.
**Causa raiz:** Arquivo é imagem de capa (JPEG renderizado como PDF), não o documento completo.
**Solução:** Ver §3.1 — Protocolo de Exceção.
**Status:** ✅ Protocolo formalizado.

---

### 3.1 — Protocolo de Exceção: Arquivo Original Inutilizável

**Condições de ativação:** zero blocos em todas as páginas; OCR < 50 palavras; documento de 1 página com dimensões de capa; inspeção visual confirma capa/logo sem corpo textual.

**Passos obrigatórios:**
1. Registrar limitação nas Observações da Conversão
2. Buscar documento na fonte oficial (DOI → portal do órgão → repositório citado)
3. Verificar autenticidade — apenas fontes oficiais aceitas
4. Atualizar `arquivo_original` (manter nome local) e `fonte_externa` (URL real de extração)
5. Não prosseguir sem conteúdo verificável
6. Se fonte oficial inacessível: status "Conversão Pendente — Fonte Indisponível"

---

## 4. Etapa de Auto-Verificação

| Critério | Verificação |
|---|---|
| Front Matter válido | Parse YAML + presença de todos os campos obrigatórios |
| Seções obrigatórias | Presença das 5 seções por header `#` |
| Conteúdo não vazio | Seção de conteúdo > 200 caracteres |
| Marcadores jurídicos presentes | Regex sobre o conteúdo — ausência total = falha |
| Ausência de artefatos | Busca por padrões de artefato no conteúdo |
| Quebras espúrias | > 3 linhas consecutivas < 40 caracteres = alerta |
| Assinatura presente | Detecção de data no padrão `SIG_PATTERN` |

**Resultados:** PASSOU → "Conversão Completa" | ALERTA → ressalva | FALHOU → correção autônoma ou registro

---

## 5. Organização dos Arquivos de Saída

```
governanca-ses-df/
├── WORKFLOW-ESPECIFICACAO.md
├── 01-legislacao-federal/
├── 02-legislacao-distrital/
├── 03-portarias-ministeriais/
├── 03-resolucoes-cfm/
├── 04-resolucoes-anvisa/
└── 05-referencias-internacionais/
```

---

## 6. Convenção de Nomenclatura de Arquivos

```
[TIPO]-[ORGAO]-[NUMERO]-[ANO]-[DESCRICAO-CURTA].md
```

Exemplos: `LEI-FEDERAL-12965-2014-MARCO-CIVIL-DA-INTERNET.md`, `RESOLUCAO-CFM-2314-2022-TELEMEDICINA.md`

---

## 7. Convenção de Versão de Transcrição

| Versão | Significado |
|---|---|
| `1.0` | Primeira transcrição |
| `1.1`, `1.2`... | Correções de extração |
| `2.0` | Reprocessamento completo |

---

## 8. Visão de Automação (Roadmap)

- **Fase atual:** Assistida com controle de qualidade humano
- **Próxima fase:** Assistida com auto-verificação (PASSOU → entrega automática; ALERTA/FALHOU → revisão humana)
- **Fase futura:** Sequencial autônoma com relatório consolidado
- **Condições para progressão:** Taxa PASSOU ≥ 95% em ≥ 10 documentos consecutivos; nenhum P01–P07 reincidente

---

## 9. Status dos Documentos

### Fase 1 — Regulamentações nacionais

| # | Documento | Arquivo | Status |
|---|---|---|---|
| 01 | Lei 12.965/2014 — Marco Civil | `LEI-FEDERAL-12965-2014-MARCO-CIVIL-DA-INTERNET.md` | ✅ v2.3 |
| 02 | Lei 13.709/2018 — LGPD | `LEI-FEDERAL-13709-2018-LGPD.md` | ✅ v2.3 |
| 03 | Lei 13.853/2019 — Altera LGPD/ANPD | `LEI-FEDERAL-13853-2019-ALTERA-LGPD-CRIA-ANPD.md` | ✅ v1.3 |
| 04 | Lei 13.787/2018 — Digitalização | `LEI-FEDERAL-13787-2018-DIGITALIZACAO-DOCUMENTOS.md` | ✅ v1.3 |
| 05 | Lei 13.989/2020 — Telemedicina | `LEI-FEDERAL-13989-2020-TELEMEDICINA.md` | ✅ v1.2 |
| 06 | Lei GDF 7.215/2023 — Telemedicina DF | `LEI-DISTRITAL-GDF-7215-2023-TELEMEDICINA-DF.md` | ✅ v1.2 |
| 07 | Portaria GM/MS 3.632/2020 — ESD28 | `PORTARIA-GMMS-3632-2020-ESTRATEGIA-SAUDE-DIGITAL-ESD28.md` | ✅ v1.2 |
| 08 | Portaria GM/MS 3.233/2024 — SUS Digital | `PORTARIA-GMMS-3233-2024-PROGRAMA-SUS-DIGITAL.md` | ✅ v1.2 |
| 09 | Portaria GM/MS 3.727/2024 — INMSD | `PORTARIA-GMMS-3727-2024-INMSD.md` | ✅ v1.1 |
| 10 | Portaria GM/MS 7.495/2025 | `PORTARIA-GMMS-7495-2025-SUS-DIGITAL-AGORA-TEM-ESPECIALISTAS.md` | 🔄 Pendente |
| 11 | Portaria SESDF 513/2022 | `PORTARIA-SESDF-513-2022-SERVICO-TELEMEDICINA-DF.md` | 🔄 Pendente |
| 12 | Portaria SAES/MS 2.326/2024 | `PORTARIA-SAESMS-2326-2024-TELESSAUDE-TABELA-SUS.md` | 🔄 Pendente |
| 13 | Resolução CFM 2.227/2018 | `RESOLUCAO-CFM-2227-2018-TELEMEDICINA.md` | 🔄 Pendente |
| 14 | Resolução CFM 2.314/2022 | `RESOLUCAO-CFM-2314-2022-TELEMEDICINA.md` | 🔄 Pendente |
| 15 | Resolução CFM 2.454/2026 | `RESOLUCAO-CFM-2454-2026-INTELIGENCIA-ARTIFICIAL.md` | 🔄 Pendente |
| 16 | RDC ANVISA 657/2022 | `RDC-ANVISA-657-2022-SAMD.md` | 🔄 Pendente |
| 17 | IN SES-DF 01/2023 | `IN-SESDF-01-2023-TELEMEDICINA-DF.md` | 🔄 Pendente |

### Fase 2 — Referências internacionais (10 arquivos bilíngues EN+PT)

| # | Documento | Status |
|---|---|---|
| 18-19 | IMDRF SaMD Key Definitions 2013 (EN+PT) | ✅ v1.0 |
| 20-21 | IMDRF SaMD Risk Categorization 2014 (EN+PT) | ✅ v1.0 |
| 22-23 | EU AI Act Overview 2024 (EN+PT) | ✅ v1.0 |
| 24-25 | OECD Health Data Governance 2025 (EN+PT) | ✅ v1.0 |
| 26-27 | PAHO 8 Principles Digital Transformation 2021 (EN+PT) | ✅ v1.0 |

### Fase 3 — Migração GitHub (este repositório)
✅ Repositório `governanca-ses-df` criado em 2026-06-02
🔄 Upload dos 28 arquivos .md — pendente

---

*Documento mantido pelo pipeline de transcrição. Atualizar a cada nova solução (§3) e mudança de status (§9).*
