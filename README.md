# Projeto Final — ACID na prática em SGBDs Relacionais

Este repositório organiza o passo a passo para produzir um trabalho **bem detalhado** sobre como um **SGBD relacional** implementa, na prática, os princípios do **ACID**: **Atomicidade, Consistência, Isolamento e Durabilidade**.

**Entrega:** até **24/02/2026** (pode ser antes)  
**Formato do relatório:** artigo SBC (~10 páginas)  
**Trabalho:** em dupla + apresentação oral

---

## 1) Objetivo e entregáveis

### 1.1 Objetivo
Investigar como um SGBD relacional real implementa suporte a **transações** e garante cada propriedade do **ACID**. A investigação deve ser sustentada por:
- **Análise de código-fonte** (módulos/arquivos/funções/fluxos)
- **Arquitetura** (diagrama do caminho transacional)
- **Evidências** (trechos curtos de código, tabelas e rastreabilidade)
- **Micro-experimentos** (opcional, mas recomendado) para validar comportamento observado

### 1.2 Entregáveis mínimos
1) **Relatório técnico (~10 páginas, SBC)** contendo:
- Contextualização do SGBD e versão analisada (commit/tag)
- Visão geral da arquitetura relevante para transações
- Caminhos de execução (BEGIN/COMMIT/ROLLBACK)
- Seções por propriedade do ACID com rastreabilidade:
  - conceito → mecanismo → módulo/arquivo → função → efeito
- Discussão e limitações (trade-offs)

2) **Apresentação oral (slides)**
- Síntese dos achados (arquitetura + ACID “na prática” + evidências principais)

---

## 2) Escopo e escolhas iniciais (Dia 1)

### 2.1 Escolha do SGBD (definir hoje)
Selecionar **1** SGBD principal para aprofundar (recomendado), com no máximo um segundo apenas para comparação pontual.

**Exemplos comuns:**
- PostgreSQL
- MySQL (foco no InnoDB)
- MariaDB
- SQLite

> Critérios: disponibilidade de código-fonte, documentação técnica, facilidade de navegar nos módulos de transações, e comunidade ativa.

### 2.2 Definir o recorte do que será analisado
**Escopo recomendado (suficiente para um trabalho forte):**
- Transações: `BEGIN`, `COMMIT`, `ROLLBACK`
- Controle de concorrência: **locks** e/ou **MVCC**, níveis de isolamento
- Logging: WAL/redo/undo, checkpoints
- Recovery: crash recovery e retorno a um estado consistente
- Persistência: flush/fsync e garantias do commit

**Fora de escopo (evitar expansão desnecessária):**
- Replicação e alta disponibilidade (citar apenas se necessário)
- Otimizador de consultas
- Engines alternativas (no MySQL, não misturar engines; foco no InnoDB)

---

## 3) Estrutura do repositório (Dia 1–2)

Sugestão de organização:

/notes/ # anotações por tema (ACID, módulos, fluxos)
/code-map/ # mapa de arquivos, funções e estruturas relevantes
/figures/ # diagramas (arquitetura, commit path, etc.)
/experiments/ # scripts SQL e resultados de micro-experimentos
/report/ # artigo SBC (LaTeX + bibliografia)
/slides/ # apresentação (PPTX/Google Slides export + assets)

---

## 4) Preparação do ambiente e fixação de versão (Dia 1–2)

### 4.1 Clonar o repositório do SGBD e fixar versão
- Clonar o repositório oficial
- Escolher uma **tag/release/commit**
- Registrar no relatório:
  - versão, hash do commit e data
  - SO, compilador/ambiente (se compilarem)
  - comandos principais usados (build/run opcional)

### 4.2 Kit de engenharia reversa (ferramentas)
Recomendado:
- `ripgrep (rg)` para busca rápida
- `ctags`/`cscope`/LSP (ex.: clangd) para navegar por símbolos
- IDE/editor com “go to definition”
- ferramenta de diagrama (draw.io, Excalidraw, Mermaid)

---

## 5) Metodologia (o “como vamos investigar”)

A metodologia é dividida em etapas com artefatos concretos. A ideia é produzir rastreabilidade suficiente para que o leitor consiga **validar** o que foi afirmado.

---

## 6) Etapa 1 — Construir o “Mapa ACID → Mecanismos” (Dia 2)

Antes de ler o código a fundo, construir uma matriz que guiará a investigação.

### 6.1 Matriz inicial (rascunho)
- **Atomicidade**
  - rollback/undo/journal
  - commit como ponto atômico
- **Consistência**
  - constraints + invariantes internas
  - recovery para estado válido
  - ordem de escrita (ex.: log antes de dados)
- **Isolamento**
  - locks e/ou MVCC
  - snapshots/versões
  - conflitos e níveis de isolamento
- **Durabilidade**
  - WAL/redo + flush/fsync
  - checkpoints
  - garantia pós-crash

### 6.2 Artefato esperado
Criar o arquivo:  
`/notes/acid_mapa_mecanismos.md`

> Comentário: esta matriz vira o “roteiro” do trabalho. Nas próximas etapas, vocês apenas preenchem com evidências do SGBD escolhido.

---

## 7) Etapa 2 — Rastrear o caminho de execução de transações (Dia 3–5)

### 7.1 Objetivo
Responder com precisão:
- O que acontece quando o usuário executa **BEGIN**?
- O que acontece internamente no **COMMIT**?
- Como o **ROLLBACK** desfaz mudanças?
- Em que momento o commit se torna “durável”?

### 7.2 Procedimento
1) Localizar no código os pontos de entrada para:
   - begin/commit/rollback
2) Seguir as chamadas até identificar:
   - mudança de estado da transação
   - escrita de log (WAL/redo/undo)
   - flush/sync
   - liberação de locks/versões
3) Registrar:
   - arquivos e funções (com caminho completo)
   - estruturas de dados relevantes
   - condições importantes (branching)

### 7.3 Artefatos esperados
- `code-map/commit_path.md`  
  Lista “Top 10 funções” com 1 linha de propósito cada.
- `figures/commit_path_diagram.*`  
  Diagrama do fluxo de COMMIT (caixas e setas).

> Comentário: este é o núcleo do relatório. Um bom trabalho não precisa “entender todo o SGBD”, mas precisa identificar com clareza os **pontos de decisão**: onde loga, onde sincroniza, onde marca commit e onde desfaz.

---

## 8) Etapa 3 — Evidências por propriedade do ACID (Semana 2)

A partir do caminho transacional, dividir a análise em quatro trilhas.

---

### 8.1 Atomicidade — checklist de evidências
**Perguntas-guia**
- Qual mecanismo garante “tudo ou nada” em caso de falha?
- Como o ROLLBACK é implementado?
- O COMMIT tem um “ponto atômico”? Qual?

**O que mapear**
- rotinas de abort/rollback
- estruturas de undo/journal (se existirem)
- interação com logging (ex.: compensação/undo vs redo)

**Artefato**
- `notes/acid_atomicidade.md` (com “mecanismo → arquivo/função → descrição”)

> Comentário: atomicidade costuma ser a parte onde o código aparece com maior clareza (rotinas de abort + estruturas de undo/journal).

---

### 8.2 Consistência — checklist de evidências
**Perguntas-guia**
- Consistência aqui significa constraints SQL? invariantes internas? ambos?
- Como o recovery garante retorno a um estado consistente?
- Existe uma regra de ordenação do tipo “log antes de dados” (WAL rule)?

**O que mapear**
- validação e aplicação de constraints (pontos principais)
- invariantes internas do storage/index/catalog (nível que fizer sentido para o recorte)
- fluxo de recovery e aplicação de log

**Artefato**
- `notes/acid_consistencia.md`

> Comentário: consistência frequentemente é “emergente” de constraints + logging + recovery + ordenação correta.

---

### 8.3 Isolamento — checklist de evidências
**Perguntas-guia**
- O SGBD usa MVCC, locks ou modelo híbrido?
- Onde são implementados snapshots/versões?
- Como lida com conflitos (write-write), deadlocks, phantoms?

**O que mapear**
- lock manager (ou equivalente)
- MVCC: estruturas de versão, timestamps, snapshots
- níveis de isolamento e comportamento por nível

**Artefato**
- `notes/acid_isolamento.md`

> Comentário: isolamento tende a ser a parte mais arquitetural. Um diagrama de “leitura vê snapshot X / escrita cria versão Y / commit valida” costuma elevar a qualidade do trabalho.

---

### 8.4 Durabilidade — checklist de evidências
**Perguntas-guia**
- Quando o commit se torna durável (garantido após crash)?
- Onde o log é gravado (WAL/redo) e onde ocorre flush/fsync?
- Como checkpoints reduzem o tempo de recovery?

**O que mapear**
- gerência de WAL/redo (escrita, buffers, flush)
- política de fsync/flush e condições
- checkpoints (quando e como)
- fluxo de crash recovery

**Artefato**
- `notes/acid_durabilidade.md`

> Comentário: sem apontar o momento do flush/fsync, a argumentação de durabilidade fica fraca. É essencial mapear “nesta etapa o log é forçado a disco”.

---

## 9) Etapa 4 — Micro-experimentos de validação (Semana 3)

Mesmo sendo um trabalho de análise de código, **1–2 micro-experimentos** fortalecem muito o relatório.

### 9.1 Exemplos de micro-experimentos
1) **Isolamento**: duas sessões concorrentes demonstrando fenômenos (dependendo do nível)
2) **Durabilidade**: commit + reinício do servidor/processo (ou simulação de crash, quando possível) e verificação pós-restart
3) **Atomicidade**: transação que altera múltiplas tabelas e falha no meio (resultado deve ser “nada aplicado”)

### 9.2 Artefatos esperados
- `experiments/exp01_isolation.sql` + `experiments/exp01_result.md`
- `experiments/exp02_durability.sql` + `experiments/exp02_result.md`
- (opcional) scripts de automação e logs

> Comentário: experimentos não precisam ser complexos. O objetivo é “ancorar” comportamento observado com os mecanismos descritos no código.

---

## 10) Produção do relatório (Semana 4)

### 10.1 Estrutura recomendada (~10 páginas)
1. **Introdução** (0,8–1 pág): motivação, problema, objetivo, método
2. **Fundamentos** (1–1,5 pág): ACID e transações (objetivo e conciso)
3. **Visão geral do SGBD** (1–1,5 pág): arquitetura relevante + componentes transacionais
4. **Implementação de transações** (1–1,5 pág): caminho BEGIN/COMMIT/ROLLBACK + diagrama
5. **ACID na prática** (4–5 pág): 1 subseção por propriedade, com evidências de código
6. **Micro-experimentos e discussão** (0,8–1 pág)
7. **Conclusão e trabalhos futuros** (0,5 pág)
8. **Referências**

### 10.2 Regras práticas para “cara de artigo”
- Cada propriedade do ACID deve conter:
  - mecanismo(s) principais
  - onde está no código (arquivo/função/estrutura)
  - fluxo resumido (passos numerados)
  - implicações e limitações (trade-offs)
- Trechos de código: curtos, comentados e diretamente relevantes.
- Diagramas: pelo menos (1) commit path e (1) visão de concorrência/log.

### 10.3 Artefatos
- `report/` com template SBC (LaTeX) e bibliografia
- figuras em `figures/` referenciadas no texto
- tabela “ACID → módulos” como resumo

---

## 11) Produção dos slides e apresentação (Semana 5)

### 11.1 Roteiro recomendado (10–12 slides)
1) Problema e objetivo  
2) SGBD escolhido e versão  
3) Arquitetura transacional (diagrama)  
4) Caminho do COMMIT (diagrama)  
5) Atomicidade (mecanismo + evidência)  
6) Consistência (mecanismo + evidência)  
7) Isolamento (mecanismo + evidência)  
8) Durabilidade (mecanismo + evidência)  
9) Micro-experimento 1 (resultado)  
10) Micro-experimento 2 (resultado)  
11) Principais achados (bullet points)  
12) Conclusão  

> Comentário: na fala, reduzir teoria genérica e enfatizar “o que encontramos no código” e “o que isso implica”.

---

## 12) Divisão de trabalho em dupla (modelo eficiente)

### Pessoa A — Logging/Recovery
- Durabilidade (WAL/redo, flush/fsync, checkpoint)
- Consistência ligada ao recovery

### Pessoa B — Concorrência/Transação
- Isolamento (MVCC/locks/snapshots)
- Atomicidade ligada a rollback/undo/journal

### Ambos
- Introdução, visão geral do SGBD, caminho do COMMIT
- Revisão final, formatação SBC, slides e ensaio

---

## 13) Cronograma sugerido até 24/02/2026

> Referência: hoje **20/01/2026**.

### Semana 1 — 20/01 a 26/01
- [ ] Escolher SGBD + versão (tag/commit)
- [ ] Montar repo e estrutura de pastas
- [ ] Produzir matriz ACID → mecanismos (`notes/acid_mapa_mecanismos.md`)
- [ ] Mapear rascunho do “commit path” (primeiro diagrama)

### Semana 2 — 27/01 a 02/02
- [ ] Evidências de Atomicidade e Durabilidade (arquivos/funções)
- [ ] Rascunho de 2 seções do relatório (texto inicial)

### Semana 3 — 03/02 a 09/02
- [ ] Evidências de Isolamento e Consistência
- [ ] Micro-experimentos (scripts + resultados)
- [ ] Fechar tabela “Top funções/módulos”

### Semana 4 — 10/02 a 16/02
- [ ] Primeira versão completa do relatório
- [ ] Diagramas finais
- [ ] Revisão técnica (coerência, rastreabilidade, referências)

### Semana 5 — 17/02 a 24/02
- [ ] Slides finais + ensaio
- [ ] Revisão final de formatação (SBC) e bibliografia
- [ ] Entrega final até **24/02/2026**

---

## 14) Checklist de qualidade (para garantir “bem detalhado”)

### Rastreabilidade e evidência
- [ ] Cada letra do ACID tem:
  - [ ] 1–2 mecanismos principais
  - [ ] 1 pseudo-fluxo ou diagrama (quando aplicável)
  - [ ] 3–6 referências diretas a arquivos/funções/estruturas
- [ ] Caminho BEGIN/COMMIT/ROLLBACK documentado (diagrama + lista de funções)

### Validação
- [ ] Pelo menos 2 micro-experimentos documentados (scripts + resultados + interpretação)
- [ ] Versão do código e ambiente registrados (reprodutibilidade)

### Qualidade de texto e apresentação
- [ ] Conclusão traz limitações reais e trade-offs
- [ ] Referências coerentes e suficientes (doc oficial + textos técnicos)
- [ ] Slides focam em achados e não em definições

---

## 15) Templates úteis (copiar/colar)

### 15.1 Template: Mapa de evidências (por propriedade)
Criar para cada propriedade (`notes/acid_*.md`):

| Conceito | Mecanismo | Arquivo/Módulo | Função/Estrutura | O que faz | Observações |
|---|---|---|---|---|---|
| Atomicidade | Undo log | path/to/file.c | func_name() | desfaz alterações | condições/branches |

### 15.2 Template: Registro de leitura de código
`code-map/reading_log.md`:

- Data:
- Trecho analisado (pasta/arquivo):
- Símbolos encontrados:
- Fluxo resumido (passos):
- Dúvidas e próximos passos:

### 15.3 Template: Micro-experimento
`experiments/expXX_result.md`:

- Objetivo:
- Ambiente (versão/config):
- Script:
- Passo a passo de execução:
- Resultado observado:
- Interpretação (conexão com mecanismo/código):
- Limitações:

---

## 16) Próxima decisão (obrigatória)
Definir:
- **SGBD escolhido**:
- **Versão/commit**:
- **Divisão A/B**:

A partir dessas escolhas, o repositório pode evoluir com um “mapa cirúrgico” de diretórios e arquivos-alvo do SGBD selecionado.
