# Spec — vynamp-restore · restore

| Campo | Valor |
|---|---|
| Módulo | `vynamp-restore` |
| Crate(s) | `vynamp-restore` (módulo `restore`; crate único até uma TASK justificar divisão) |
| Tipo | app |
| Versão da spec | 0.2.0 |
| Data | 2026-11-11 |
| Estado | **rascunho autorado, não aprovado** — §3 a §6, §9 e §10 escritos a partir de `reference/tools/01-vynamp-restore.md` v0.1.1, ADR-013, ADR-015 e TASK-E18-01. A aprovação é do mantenedor, por `GOVERNANCE.md` |
| Dono | mantenedor do `vynamp-restore` |
| Depende de | `vynamp-engine`, `vynamp-dsp` |
| É dependido por | ninguem |

> §1, §2 e §7 foram decididos em `01-arvore-final.md` §2 e valem como norma.
> §3 a §6, §9 e §10 foram autorados em 2026-11-11 a partir de
> `reference/tools/01-vynamp-restore.md`, que continua sendo a fonte normativa do
> motor, da cadeia e dos limites. Estar escrito não é estar aprovado: até o mantenedor
> aprovar, nenhum número aqui é contrato, e toda TASK que dependa dele continua
> `planned`. A parte de IA (pós-1.0) vive em `ia.md`.
>
> **A fonte normativa é o repositório principal, não este arquivo.** Ver
> [`specs/README.md`](README.md) para o documento canônico deste módulo.

## 1. Propósito

App desktop, em processo separado (ADR-013 §1), de restauracao de audio com diagnostico de defeitos.

**Não faz:** Nao converte em lote, nao toca audio, nao embute modelo de IA no instalador.

## 2. Fronteira

| Permitido | Proibido |
|---|---|
| vynamp-engine, vynamp-dsp | depender do player ou do converter; baixar modelo sem clique explicito do usuario |

Plataformas: Windows, Linux e macOS. **Não há restore no Android nem no iOS**
(decisão de 2026-11-03: o app móvel é só player).

## 3. API pública

```rust
/// Estado de restauração v1. Fora de faixa é recusado, nunca saturado.
pub struct RestoreState {
    pub version: u32,            // 1
    pub duration: f64,           // 0..=86_400 s
    pub enabled: bool,           // padrão false; false força Monitor::Original
    pub monitor: Monitor,        // padrão Treated
    pub scope: Scope,
    pub tools: Tools,
    pub regions: Vec<Region>,    // ≤ 128, sem sobreposição
    pub diagnostics: Option<Diagnostics>,
    pub engine_version: String,
}

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum Monitor { Original, Treated, Removed } // Removed = original − tratado; troca com crossfade ~6 ms

#[derive(Clone, Copy, PartialEq, Eq, Debug)]
pub enum Scope { All, Regions }

pub struct Tools {
    pub dc: bool,
    pub hum: u8,                 // 0 | 50 | 60
    pub hum_adaptive: bool,      // com hum = 0 aplica 50 e 60
    pub gate: bool,
    pub gate_threshold_db: f32,  // −72..=−12, padrão −45
    pub click: bool,
    pub click_threshold: f32,    // 0.03..=0.9, padrão 0.2
    pub rumble_hz: f32,          // 0..=80
    pub decrackle: f32,          // 0..=0.5
    pub decrackle_mode: DecrackleMode, // Soft | Strong, padrão Strong
    pub decrackle_cutoff_hz: f32,      // 8_000..=22_000, padrão 18_000
    pub declip_db: f32,          // −12..=0
    pub deartifact: f32,         // 0..=1
    pub bwe: f32,                // 0..=1; síntese, rotulada como estimativa
    pub bwe_cutoff_hz: f32,      // 8_000..=22_000, padrão 16_000
}

pub struct Region {
    pub id: String,              // ≤ 80 chars, único
    pub start: f64,
    pub end: f64,                // end − start ≥ 0,01 s
    pub gain_db: f32,            // −90..=0; ≤ −80 é silêncio
    pub fade: f64,               // 0,005..=1 s, limitado a (end − start)/2
    pub kind: RegionKind,        // só Between; Overlay não é portado
}

pub fn validate(state: &RestoreState) -> Result<(), RestoreError>;

/// Cadeia em ordem fixa (herdada do MVP rev.38; mudar exige divergência ADR-013).
pub struct Chain { /* privado */ }
impl Chain {
    pub fn new(state: &RestoreState, sample_rate: u32, channels: u16) -> Result<Self, RestoreError>;
    /// Latência em amostras; o original é atrasado por ela para alinhar.
    pub fn latency(&self) -> usize;
    /// f32 intercalado, sem alocar no caminho de processamento.
    pub fn process(&mut self, block: &mut [f32]);
}

pub struct Diagnostics {
    pub duration: f64,
    pub sample_rate: u32,
    pub channels: u16,
    pub effective_nyquist: f32,  // limite a −60 dB
    pub defects: Vec<Defect>,
    pub chain: Vec<String>,
    pub summary: String,
}

pub struct Defect {
    pub kind: DefectType,
    pub severity: Severity,      // Mild | Moderate | Severe
    pub evidence: String,
    pub suggestion: String,
    pub fix: Option<ToolsPatch>, // None para hiss, discontinuity e skip
    pub examples: Vec<f64>,      // ≤ 6 instantes
}

#[non_exhaustive]
pub enum DefectType {
    Clipping, TruePeak, Click, Crackle, Hum50, Hum60, Hiss,
    Rumble, Discontinuity, Skip, DcOffset, BandLimit,
}

pub fn analyze(path: &Path, cancel: &CancelToken) -> Result<Diagnostics, RestoreError>;
/// Aplica só Moderate/Severe com fix; liga o tratamento e põe o monitor em Treated.
pub fn apply_recommended(state: &mut RestoreState) -> Vec<AppliedFix>;

pub struct Preset { pub id: String, pub name: String, pub description: String, pub tools: ToolsPatch }
pub const BUILTIN_PRESETS: [&str; 5] = [
    "builtin-scratched-cd", "builtin-vinyl", "builtin-cassette",
    "builtin-phone-recording", "builtin-low-mp3",
];

pub struct ExportOptions {
    pub format: ExportFormat,    // Flac | Wav16 | Wav24 | Wav32 | WavF32 | Aiff | Alac
    pub limiter: bool,           // padrão true: −1,0 dBTP, lookahead 5 ms, 4×, margem 0,5 dB
    pub normalize: bool,         // padrão false: pico −1 dBFS
    pub metadata: bool,          // padrão true
    pub overwrite: bool,         // padrão false
}
pub fn export(src: &Path, dst: &Path, state: &RestoreState, opts: &ExportOptions, cancel: &CancelToken)
    -> Result<ExportReport, RestoreError>;

/// Projeto `.vynamp-restore`: JSON ≤ 256 KiB, até 12 rascunhos.
pub struct Project { pub format: String, pub version: u32, pub source: SourceRef, pub engine_version: String, pub state: RestoreState }
pub struct SourceRef { pub name: String, pub fingerprint: String } // nome + tamanho + mtime
impl Project {
    pub fn import(json: &[u8], source: &Path) -> Result<Self, RestoreError>; // inválido = recusa inteira
    pub fn export(&self) -> Result<Vec<u8>, RestoreError>;
}

#[derive(Clone, Default)]
pub struct CancelToken { /* Arc<AtomicBool> */ }
```

### CLI

`vynamp-restore [opções] <caminhos...>` — `--analyze`, `--list-defects`,
`--export <dir>`, `--format flac|wav|wav-f32|aiff|alac`, `--no-metadata`,
`--overwrite`, `--version`, `--help`. Arquivos entram na ordem recebida; nada é
reordenado nem descartado em silêncio. Instância única (TASK-E16-02): uma segunda
chamada entrega os arquivos por IPC local autenticado.

| Código | Significado |
|---|---|
| 0 | ok |
| 1 | erro de uso |
| 2 | ilegível ou não suportado |
| 3 | erro de escrita |
| 4 | cancelado |
| 5 | falha interna do motor |

### Cadeia (ordem fixa)

1. decrackle AR, 2 passes
2. passa-baixa fase linear 63 taps, Kaiser β = 6, corte min(cutoff, 0,4·fs)
3. deartifact STFT
4. bwe
5. click
6. dc, 5 Hz
7. rumble, Butterworth 2ª ordem
8. hum: notches na fundamental e harmônicos 2 e 3, Q = 30
9. gate, ataque 3 ms, release 120 ms
10. declip, joelho tanh em −1 dBFS
11. regiões e ganho: `1 + (alvo − 1)·fade`, cosseno elevado

### Limites

| Item | Limite |
|---|---|
| Duração | 24 h |
| Canais | 32 |
| Memória do motor | 512 MiB (processamento em blocos) |
| Regiões | 128 |
| Profundidade de desfazer | 60 |
| Projeto | 256 KiB, 12 rascunhos |
| Lote | 32 arquivos por fila, 8 por rodada, sequencial |
| Metadados | 16 MiB por bloco, 1024 chars por campo |

## 4. Requisitos

| ID | Requisito | REQ de produto | TASK | Teste |
|---|---|---|---|---|
| RESTORE-REQ-01 | `validate` recusa qualquer valor de `RestoreState`, `Tools` ou `Region` fora das faixas de §3, nomeando o campo; nunca satura — planned (M3) | estado validado | E18-01 | — (RESTORE-AC-01 planejado) |
| RESTORE-REQ-02 | A cadeia roda na ordem fixa de §3 e expõe `latency()`; o original é atrasado pela latência para alinhar — planned (M3) | cadeia determinada | E18-01 | — (RESTORE-AC-02 planejado) |
| RESTORE-REQ-03 | Processamento determinístico byte a byte: sem aleatoriedade, sem relógio, sem paralelismo que mude a ordem — planned (M3) | reprodutível | E18-01 | — (RESTORE-AC-03 planejado) |
| RESTORE-REQ-04 | `analyze` detecta os 12 `DefectType` com severidade, evidência e até 6 exemplos; hiss, discontinuity e skip vêm sem fix; há um único detector de click — planned (M3) | diagnóstico | E18-01 | — (RESTORE-AC-04 planejado) |
| RESTORE-REQ-05 | Correções são propostas, nunca aplicadas ao abrir; “Aplicar recomendadas” aplica só Moderate/Severe com fix, e cada correção pode ser desfeita e fica registrada — planned (M3) | sem efeito oculto | E18-01 | — (RESTORE-AC-05 planejado) |
| RESTORE-REQ-06 | Monitor Original/Treated/Removed troca com crossfade ~6 ms; `enabled = false` força Original; Removed é original − tratado — planned (M3) | comparação honesta | E18-01 | — (RESTORE-AC-06 planejado) |
| RESTORE-REQ-07 | Exportação grava em temporário no mesmo volume → fsync → relê e valida (contêiner, taxa, canais, duração) → renomeia; em falha o temporário é removido e a fonte nunca é tocada — planned (M3) | escrita atômica | E18-01 | — (RESTORE-AC-07 planejado) |
| RESTORE-REQ-08 | Limitador ligado por padrão (−1,0 dBTP, lookahead 5 ms, 4×); normalização desligada por padrão; arquivo de evidência com versão do motor, parâmetros, defeitos e correções — planned (M3) | saída rotulada | E18-01 | — (RESTORE-AC-08 planejado) |
| RESTORE-REQ-09 | Projeto `.vynamp-restore` importado inválido é recusado inteiro; projeto de outro arquivo (fingerprint diferente) é recusado — planned (M3) | projeto seguro | E18-01 | — (RESTORE-AC-09 planejado) |
| RESTORE-REQ-10 | Lote: 32 por fila, 8 por rodada, sequencial; falha de um arquivo não para a fila; cancelamento por arquivo ou pela fila inteira; relatório agregado — planned (M3) | lote previsível | E18-01 | — (RESTORE-AC-10 planejado) |
| RESTORE-REQ-11 | A CLI devolve os códigos 0–5 de §3 e preserva a ordem dos caminhos — planned (M3) | CLI estável | E18-01, E16-02 | — (RESTORE-AC-11 planejado) |
| RESTORE-REQ-12 | Presets embutidos são imutáveis; preset do usuário só ajusta as ferramentas que lista; aviso obrigatório quando o preset liga hum ou corte abaixo de 8 kHz — planned (M3) | presets honestos | E18-01 | — (RESTORE-AC-12 planejado) |
| RESTORE-REQ-13 | Metadados são lidos (ID3v2.2/2.3/2.4, LIST/INFO, `id3 `, `_PMX`, XMP) e campos desconhecidos copiados; falha ao gravar metadados não bloqueia a exportação — planned (M3) | metadados preservados | E18-01 | — (RESTORE-AC-13 planejado) |
| RESTORE-REQ-14 | BWE é rotulado como estimativa e fica indisponível quando a fonte não tem conteúdo na oitava abaixo do corte — planned (M3) | rótulo honesto | E18-01 | — (RESTORE-AC-14 planejado) |

Os rótulos da coluna “REQ de produto” são descritivos; o ID de produto correspondente
é o da TASK citada no backlog.

## 5. Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| RESTORE-AC-01 | tabela gerada das faixas de §3, com um valor de cada lado fora (ex.: gate −73 e −11 dB, 129 regiões, região de 0,009 s, regiões sobrepostas) | `validate` roda sobre cada caso | todo caso dentro passa; todo caso fora falha com `InvalidState { field }` correto e nenhum valor é ajustado |
| RESTORE-AC-02 | estado com todas as ferramentas ligadas e impulso unitário na entrada | a cadeia processa e o monitor alterna Original/Treated | a ordem registrada em `Diagnostics.chain` é a de §3 e o original sai alinhado ao tratado pela `latency()` |
| RESTORE-AC-03 | o mesmo arquivo e o mesmo estado | `export` roda duas vezes | os dois arquivos são idênticos byte a byte |
| RESTORE-AC-04 | fixtures sintéticas com um defeito cada (clipping, click, hum 50, tom musical puro em 50 Hz, dc_offset, band_limit, hiss) | `analyze` roda | cada defeito é detectado com o tipo certo; o tom musical não vira hum; hiss vem com `fix = None` |
| RESTORE-AC-05 | diagnóstico com um defeito Mild, um Moderate e um Severe, todos com fix | o arquivo é aberto e depois “Aplicar recomendadas” | ao abrir nada é aplicado; depois só Moderate e Severe aparecem em `AppliedFix`, e desfazer volta ao estado anterior |
| RESTORE-AC-06 | `enabled = false` e monitor Treated | o motor processa um bloco | a saída é o original; com `enabled = true` e Removed, a saída é original − tratado |
| RESTORE-AC-07 | exportação interrompida por falha simulada após o fsync | `export` roda | não sobra temporário, o destino não existe e a fonte tem o mesmo hash de antes |
| RESTORE-AC-08 | sinal com pico em 0 dBFS, opções padrão | `export` roda | o true peak medido é ≤ −1,0 dBTP, nenhuma normalização foi feita e o arquivo de evidência existe com os campos de §4 |
| RESTORE-AC-09 | projeto com um campo inválido e projeto de outro arquivo | `Project::import` roda | os dois são recusados inteiros, sem estado parcial |
| RESTORE-AC-10 | fila de 33 arquivos, um deles ilegível | o lote roda | o 33º é recusado na entrada, os 32 rodam em rodadas de 8, o ilegível falha sem parar a fila e o relatório agrega os demais |
| RESTORE-AC-11 | chamadas com opção inválida, com arquivo ilegível e com cancelamento | a CLI roda | os códigos de saída são 1, 2 e 4, e a ordem dos arquivos no relatório é a de entrada |
| RESTORE-AC-12 | preset do usuário que liga hum | o preset é aplicado | só as ferramentas listadas mudam e o aviso aparece; presets embutidos recusam edição |
| RESTORE-AC-13 | arquivo com ID3v2.4 e um frame desconhecido; falha simulada na gravação de metadados | `export` roda | o frame desconhecido é copiado; com a falha simulada a exportação termina e registra o aviso |
| RESTORE-AC-14 | fonte com conteúdo até 16 kHz e fonte com conteúdo só até 4 kHz, ambas com `bwe_cutoff_hz` 16 kHz | o estado com BWE é aplicado ao arquivo analisado | a primeira aceita e rotula “estimativa”; a segunda recusa com `InvalidState { field: "bwe" }` |

**Nenhum destes testes existe ainda.** O nome é o compromisso; a TASK citada em §4 é
quem os escreve, e o teste nasce vermelho antes da implementação.

### Critérios que esta spec **não** fixa

- Algoritmo interno de cada detector além do que §3 declara (os limiares de tom
  musical, 3 % e 5 %, ficam na fonte normativa).
- Layout da UI, cores e atalhos: são da casca do app (identidade `#66D9EF`).
- Qualidade perceptual da restauração: aqui só determinismo e limites são verificáveis.

## 6. Erros e cancelamento

**Erro é dado, não `panic!`.**

| Situação | Erro | Código CLI |
|---|---|---|
| Valor fora de faixa ou regiões sobrepostas | `InvalidState { field, reason }` | 1 |
| Arquivo ilegível ou formato não suportado | `Unreadable { path, reason }` | 2 |
| Falha de escrita, fsync, revalidação ou rename | `Write { path, reason }` | 3 |
| Cancelado pelo usuário | `Cancelled` | 4 |
| Limite de memória ou invariante do motor | `Engine { reason }` | 5 |
| Projeto inválido ou de outro arquivo | `ProjectRejected { reason }` | 1 |

**Cancelamento é explícito, não timeout.** `CancelToken` é verificado a cada bloco em
`analyze`, `export` e no lote; cancelar um arquivo remove seu temporário e não toca a
fonte; cancelar a fila para depois do bloco corrente.

## 7. Fora de escopo

Modelos de IA: download opt-in com SHA-256 e selecao no painel (ADR-015); nenhum modelo no instalador

## 8. Dependências

Depende de: `vynamp-engine`, `vynamp-dsp`

É dependido por: ninguem

## 9. Testes e fixtures

| Fixture | Origem | Uso |
|---|---|---|
| tabela de faixas | gerada no teste a partir de §3 | RESTORE-AC-01 |
| sinais sintéticos com um defeito cada | gerados no teste (seno, impulsos, DC, ruído com semente fixa) | RESTORE-AC-02 a RESTORE-AC-06, RESTORE-AC-08, RESTORE-AC-14 |
| arquivos pequenos com metadados | `tests/fixtures/`, gerados por script versionado | RESTORE-AC-07, RESTORE-AC-09 a RESTORE-AC-13 |

Oráculo: as faixas e a ordem de §3, o hash do arquivo para determinismo e a medição de
true peak para o limitador. Os testes com áudio são curtos; nada fica `#[ignore]`.

```
cargo test --test estado_restore --locked
cargo test --test cadeia_restore --locked
cargo test --test exportacao_restore --locked
```

## 10. Limitações conhecidas

Limitação declarada não reprova a entrega; limitação omitida reprova.

| Limitação | Efeito | Quem cobre |
|---|---|---|
| Nenhum tipo desta spec existe em `src/` | nenhum critério verificável hoje | E18-01 |
| Sem restore no Android e no iOS | usuário móvel não restaura | decisão de 2026-11-03 |
| Região `overlay` não é portada | separação de fontes fica para `ia.md` | `ia.md` |
| Edição de tags é do player | o restore só preserva metadados | TASK-E08-04 |
| Qualidade perceptual não tem oráculo automático | aceitação depende de escuta (QA-015, QA-016, QA-023, QA-024, QA-031) | QA citados |
