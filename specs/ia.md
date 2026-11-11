# Spec — vynamp-restore · ia

| Campo | Valor |
|---|---|
| Módulo | `vynamp-restore` |
| Crate(s) | `vynamp-restore` (módulo `ia`, atrás da feature Cargo `ai`; crate único até uma TASK justificar divisão) |
| Tipo | app |
| Versão da spec | 0.2.0 |
| Data | 2026-11-11 |
| Estado | **rascunho autorado, não aprovado; fase futura planejada e não prometida (pós-1.0)** — §3 a §6, §9 e §10 escritos a partir de `reference/tools/03-vynamp-restore-ia.md` v0.1.0, ADR-011 e ADR-015. A aprovação é do mantenedor, por `GOVERNANCE.md` |
| Dono | mantenedor do `vynamp-restore` |
| Depende de | `vynamp-engine`, `vynamp-dsp` |
| É dependido por | ninguem |

> §1, §2 e §7 foram decididos em `01-arvore-final.md` §2 e valem como norma.
> §3 a §6, §9 e §10 foram autorados em 2026-11-11 a partir de
> `reference/tools/03-vynamp-restore-ia.md`, que continua sendo a fonte normativa.
> Estar escrito não é estar aprovado: até o mantenedor aprovar, nenhum número aqui é
> contrato. **Nada desta spec entra antes de 1.0**, e nada começa enquanto as cinco
> condições de entrada de §10 não forem cumpridas.
>
> **A fonte normativa é o repositório principal, não este arquivo.** Ver
> [`specs/README.md`](README.md) para o documento canônico deste módulo.

## 1. Propósito

App desktop, em processo separado (ADR-013 §1), de restauracao de audio com diagnostico de defeitos.

**Não faz:** Nao converte em lote, nao toca audio, nao embute modelo de IA no instalador.

## 2. Fronteira

| Permitido | Proibido |
|---|---|
| herda a fronteira de `specs/restore.md` | herda as proibicoes de `specs/restore.md` |

Regra própria desta parte (ADR-011): o processamento em tempo real continua 100 %
próprio, sem ML. Inferência só offline, via `candle`, `ort` ou `tract`, atrás da
feature Cargo `ai`; uma build sem a feature não tem nenhuma dependência de ML.
Adotar qualquer dessas crates exige um ADR novo que emende o ADR-011.

## 3. API pública (esboço)

```rust
#[cfg(feature = "ai")]
pub mod ia {
    /// Catálogo fixo; tamanho, origem e licença são exibidos antes do download.
    pub struct ModelCatalog { pub entries: Vec<ModelEntry> }
    pub struct ModelEntry {
        pub id: String,
        pub task: ModelTask,       // SuperRes | Separation
        pub size_bytes: u64,
        pub source_url: String,
        pub license: String,
        pub sha256: [u8; 32],
    }

    /// Só em resposta a clique explícito. Grava em `%APPDATA%/vynamp/models/`
    /// (equivalente por SO), confere SHA-256 e recusa o arquivo se não bater.
    pub fn download(entry: &ModelEntry, cancel: &CancelToken) -> Result<ModelPath, IaError>;
    /// Instalação offline a partir de uma pasta escolhida pelo usuário.
    pub fn install_from_dir(dir: &Path) -> Result<Vec<ModelPath>, IaError>;
    /// Remoção explícita; nada é apagado sem pedido.
    pub fn remove(id: &str) -> Result<(), IaError>;

    #[derive(Clone, Copy, PartialEq, Eq, Debug)]
    pub enum Backend { Cpu, Cuda, Metal, DirectMl } // Cpu é o piso e o padrão

    pub struct SuperResJob {
        pub model: SuperResModel,  // AudioSrBasic | AudioSrSpeech (~1–2 GB cada)
        pub backend: Backend,
        pub cutoff_hz: f32,        // vem de band_limit do diagnóstico
        pub seed: u64,             // fixa
        pub vram_ceiling_mb: Option<u32>,
    }
    impl SuperResJob {
        /// Gera um arquivo NOVO a 48 kHz; a fonte nunca é tocada.
        pub fn run(&self, src: &Path, dst: &Path, cancel: &CancelToken)
            -> Result<SuperResReport, IaError>;
    }
}
```

### Pipeline de super-resolução

pré-processamento → codificador VAE → UNet → DDIM (~50 passos) → decodificador VAE →
vocoder 48 kHz. Roda em blocos de 10 s com 1 s de sobreposição e crossfade de
potência igual. Só a banda acima do corte é substituída, por crossover de fase linear
com transição de ~1 kHz; abaixo do corte a saída é a fonte.

### Backends

CPU é o piso e o padrão. CUDA, Metal e DirectML são opcionais; se o backend escolhido
falhar no meio, o job retoma em CPU a partir do bloco corrente. CPU usa fp32.
Bit-exato só é exigido no mesmo backend. Teto de VRAM configurável.

### Separação de fontes (dependente de reabrir `overlay`)

| Modelo | Tamanho |
|---|---|
| MDX-Net B | 30 MB |
| Kim_Vocal_2 | 50 MB |
| Demucs v3 | 80 MB |
| htdemucs-ft | 100 MB / 316 MB |

Só entra se a região `overlay` for reaberta (hoje não portada, ver `restore.md` §10).
As máscaras são aplicadas pelo kernel do `vynamp-dsp`; não há quarto modo de monitor.

## 4. Requisitos

| ID | Requisito | REQ de produto | TASK | Teste |
|---|---|---|---|---|
| IA-REQ-01 | Build sem a feature `ai` não tem dependência de ML (verificado no grafo do `cargo tree`) — planned (pós-1.0) | RSAI-01 | — (bloqueada por §10) | — (IA-AC-01 planejado) |
| IA-REQ-02 | Nenhum modelo no instalador; download só por clique explícito, com tamanho, origem e licença exibidos antes — planned (pós-1.0) | RSAI-02, RSAI-03 | — | — (IA-AC-02 planejado) |
| IA-REQ-03 | SHA-256 conferido; arquivo divergente é apagado e recusado; instalação offline e remoção explícita disponíveis; nenhuma telemetria — planned (pós-1.0) | RSAI-04, RSAI-05 | — | — (IA-AC-03 planejado) |
| IA-REQ-04 | Super-resolução substitui só a banda acima de `cutoff_hz`; abaixo dela a saída é idêntica à fonte reamostrada — planned (pós-1.0) | RSAI-06, RSAI-07 | — | — (IA-AC-04 planejado) |
| IA-REQ-05 | Mesmo modelo, semente, backend e entrada geram saída bit-exata — planned (pós-1.0) | RSAI-08 | — | — (IA-AC-05 planejado) |
| IA-REQ-06 | Paridade com a referência Python dentro da tolerância declarada no spike — planned (pós-1.0) | RSAI-09 | — | — (IA-AC-06 planejado) |
| IA-REQ-07 | Cancelamento responde em < 1 s; memória e VRAM respeitam o teto — planned (pós-1.0) | RSAI-10, RSAI-11 | — | — (IA-AC-07 planejado) |
| IA-REQ-08 | GPU é opcional; falha de GPU retoma em CPU a partir do bloco — planned (pós-1.0) | RSAI-12 | — | — (IA-AC-08 planejado) |
| IA-REQ-09 | Saída é arquivo novo a 48 kHz rotulado como gerado por IA; a fonte nunca é tocada — planned (pós-1.0) | RSAI-13 | — | — (IA-AC-09 planejado) |
| IA-REQ-10 | Aceitação perceptual por teste de escuta documentado — planned (pós-1.0) | RSAI-14 | — | — (IA-AC-10 planejado) |

Os rótulos RSAI-nn são os de `reference/tools/03-vynamp-restore-ia.md`. Não há TASK:
ela só é criada quando as condições de §10 forem cumpridas e um marco for atribuído.

## 5. Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| IA-AC-01 | o workspace sem a feature `ai` | `cargo tree -e normal` roda | nenhuma de `candle`, `ort`, `tract` aparece |
| IA-AC-02 | instalação limpa | o app abre e o painel de IA é aberto sem clique em baixar | nenhum arquivo é criado em `models/` e nenhuma conexão de rede é feita |
| IA-AC-03 | download servido com um byte alterado | `download` termina | o arquivo é apagado e o erro é `ChecksumMismatch` |
| IA-AC-04 | fonte com conteúdo até 11 kHz e `cutoff_hz` 11 kHz | `SuperResJob::run` termina | a banda abaixo de 10 kHz difere da fonte reamostrada apenas dentro da tolerância do crossover |
| IA-AC-05 | mesmo modelo, semente, backend CPU e entrada | o job roda duas vezes | as saídas são idênticas byte a byte |
| IA-AC-06 | o conjunto de referência do spike | o job roda | a diferença para a saída Python fica dentro da tolerância registrada no spike |
| IA-AC-07 | job em andamento | o cancelamento é pedido | o job para em < 1 s, sem arquivo parcial no destino |
| IA-AC-08 | backend GPU com falha simulada no bloco N | o job roda | o job retoma em CPU no bloco N e conclui |
| IA-AC-09 | qualquer fonte | o job conclui | a saída é um arquivo novo a 48 kHz com rótulo de IA nos metadados e a fonte mantém o hash |
| IA-AC-10 | protocolo de escuta definido no spike | o teste de escuta é feito | o resultado é registrado; sem registro não há aceite |

**Nenhum destes testes existe ainda.** O nome é o compromisso; a TASK que os escreve
só nasce depois das condições de §10.

### Critérios que esta spec **não** fixa

- Tolerância numérica de paridade: sai do spike de 10 s.
- Exigir GPU, processamento em tempo real, treino de modelo ou outros modelos de
  super-resolução: fora de escopo.

## 6. Erros e cancelamento

**Erro é dado, não `panic!`.**

| Situação | Erro |
|---|---|
| Download falhou ou foi interrompido | `Download { reason }` |
| SHA-256 diverge | `ChecksumMismatch { id }` |
| Modelo ausente | `ModelMissing { id }` |
| Backend indisponível ou falhou | `Backend { backend, reason }` (com retomada em CPU) |
| Teto de memória ou VRAM | `ResourceLimit { reason }` |
| Cancelado | `Cancelled` |

**Cancelamento é explícito, não timeout.** `CancelToken` é verificado entre passos
DDIM e entre blocos; cancelar remove o temporário e não deixa arquivo parcial.

## 7. Fora de escopo

Modelos de IA: download opt-in com SHA-256 e selecao no painel (ADR-015); nenhum modelo no instalador

## 8. Dependências

Depende de: `vynamp-engine`, `vynamp-dsp`

É dependido por: ninguem

## 9. Testes e fixtures

| Fixture | Origem | Uso |
|---|---|---|
| modelo de teste minúsculo | gerado no spike, com SHA-256 versionado | IA-AC-03, IA-AC-05, IA-AC-07, IA-AC-08 |
| conjunto de referência Python | saída do spike de 10 s | IA-AC-06 |
| sinais com banda limitada | gerados no teste | IA-AC-04, IA-AC-09 |

Ordem de implementação, quando liberada: spike → pré-processamento e vocoder →
VAE, UNet e DDIM → segmentação em blocos → job → UI → backends GPU.

```
cargo tree -e normal --locked
cargo test --features ai --test ia_superres --locked
```

## 10. Limitações conhecidas

Limitação declarada não reprova a entrega; limitação omitida reprova.

| Limitação | Efeito | Quem cobre |
|---|---|---|
| Condição 1 não cumprida: licenças dos modelos não verificadas | nenhum modelo pode ser listado | mantenedor |
| Condição 2 não cumprida: spike de 10 s não feito | sem tolerância de paridade nem custo medido | mantenedor |
| Condição 3 não cumprida: orçamento de download não definido | tamanho aceitável indefinido | mantenedor |
| Condição 4 não cumprida: decisão não registrada em `05-decisoes` e ADR que emenda o ADR-011 inexistente | nenhuma crate de ML pode entrar | mantenedor |
| Condição 5 não cumprida: nenhum marco atribuído | não há TASK nem prazo | mantenedor |
| Separação de fontes depende de reabrir `overlay` | IA-REQ para separação não existe ainda | decisão futura |
