# TASK-E18-01 — Entregar Vynamp Restore como ferramenta desktop de restauração

> Gerado pelo passo 37 a partir de `tasks/backlog.json`. Os campos de estado vêm do
> backlog e não devem ser editados aqui. A coluna `Then` é o requisito aprovado,
> copiado literal do backlog; `Given` e `When` foram autorados na execução.

| Campo | Valor |
|---|---|
| ID | `TASK-E18-01` |
| Módulo dono | `vynamp-restore` |
| Repo | `jimmykedlyn/vynamp-restore` |
| REQ coberto | `REQ-E18-01` |
| Critérios da spec | pendente — `specs/restore.md` §5 |
| Fase | M3 |
| Prioridade | P0 |
| Estado | planned |
| Depende de | `TASK-E08-01`, `TASK-E01-07`, `TASK-E16-03`, `TASK-E16-04`, `TASK-E16-05` |
| Bloqueia | `TASK-E19-04` |
| Modelo executor | Opus — áudio/RT, concorrência, segurança ou release |
| Esforço | 10–25 dias úteis |
| Risco | alto |

## Entrega

Ferramenta desktop `vynamp-restore` com motor de restauração próprio, diagnóstico de defeitos e exportação preservando metadados.

## Boundary

- **Não** implementa a fila persistida com checkpoints, leases e retomada do scheduler: isso é `TASK-E08-01`, dependência declarada desta TASK. A confusão é plausível porque o AC-4 e o §6 da spec falam de fila de 32 arquivos a 8 por rodada; aqui a fila é em memória, sequencial e cancelável, e sobreviver a um crash é do dono dos jobs.
- **Não** escreve decoder nem encoder próprios: a leitura é do `vynamp-decode`, com os codecs e capítulos novos em `TASK-E01-07`, e a escrita sai da lib `vynamp-encode`, que `TASK-E18-02` entrega como única fonte de codificadores e cujo registry é `TASK-E08-02`. A tentação vem do AC-2 e do AC-5, que releem o FLAC exportado; um codec local aqui reforkaria gapless e encoder delay, que já têm dono e teste.
- **Não** edita tags: escrita de metadado é `TASK-E03-08` no player — a spec §1.2 cita `TASK-E08-04`, que no backlog é dedupe, então a citação da spec está desatualizada e o ID correto é o do player. O AC-5 fala em reescrever o `VORBIS_COMMENT` porque o bloco cresce 240 bytes com o comentário de processamento; o Restore preserva os campos originais e acrescenta o registro, não corrige o que o usuário digitou.
- **Não** integra com o shell nem com menus de contexto: associação e menu no Windows são `TASK-E16-03`, MIME e abertura externa no Linux são `TASK-E16-04`, document types e Quick Actions no macOS são `TASK-E16-05`, todas dependências desta TASK. O `--analyze` e o `--list-defects` são a porta que esses menus chamam, não o registro do shell.
- **Não** implementa owner único de instância nem o IPC local autenticado: isso é `TASK-E16-02`. O §1.3 da spec descreve a segunda chamada entregando os caminhos à janela já aberta, o que parece parte do contrato do Restore; a TASK entrega o binário que aceita caminhos, e a exclusividade de instância tem dono no player.
- **Não** entrega instalador, pacote por plataforma nem assinatura: NSIS e portable são `TASK-E15-06`, AppImage e DEB são `TASK-E15-07`, DMG e Universal2 são `TASK-E15-08`. "Ferramenta desktop" no deliverable sugere instalador, mas aqui o artefato é o binário e o seu critério.

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e18-01-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E18-01-AC-1` | `fixtures/clipping-16-44100.flac`, FLAC 16 bits/44,1 kHz de 40 s (1 764 000 amostras) com três trechos de clipping duro somando 4 800 amostras em 0 dBFS, DR medido de 6 dB e 900 amostras acima do teto perdidas em cada canal, e um diretório de saída vazio ao lado do original | o `cargo run --release -- --analyze --list-defects fixtures/clipping-16-44100.flac` imprime o diagnóstico, a confirmação recebe a escolha explícita de ganho de −3 dB e `--export <dir>` grava o FLAC novo ao lado, com `sha256sum` da origem capturado antes e depois do comando | A fonte nunca é sobrescrita sem confirmação |
| `TASK-E18-01-AC-2` | `fixtures/tape-hiss-16-44100.flac`, FLAC 16 bits/44,1 kHz de 60 s com ruído 1/f a −42 dBFS somado sobre um trecho de repouso, um bloco `VORBIS_COMMENT` com TITLE, ARTIST e os marcadores de início e fim de cada faixa, e um projeto com `gate: true` e `threshold: −40`, o limiar acima do ruído e o único par de parâmetros de ruído que o §3.2 da spec oferece — não existe denoise na 1.0 | a exportação roda com esse projeto para arquivo novo enquanto um observador anota, no diretório de destino, o aparecimento e o sumiço do temporário, e a saída é medida em três eixos: temporário no mesmo volume seguido do `rename`, igualdade campo a campo do `VORBIS_COMMENT` e `sha256sum` da origem antes e depois | escrita atômica com validação pós-escrita preserva origem e metadados |
| `TASK-E18-01-AC-3` | `fixtures/declared-1176000.flac`, FLAC 16 bits/44,1 kHz que declara 1 176 000 amostras, o mesmo PCM gerado por um encoder de referência, e um diretório de saída cujo arquivo de destino já existe com conteúdo diferente e `sha256sum` conhecido | o `vynamp-restore` é chamado com `--export` apontando para o diretório que já contém o destino com conteúdo diferente, o usuário responde sobrescrever ou abortar antes de qualquer escrita, e o `sha256sum` do destino, o código de saída e o relatório de `--list-defects` são lidos antes e depois do comando | defeitos reportados têm severidade e evidência |
| `TASK-E18-01-AC-4` | um diretório `/tmp/cinco-flacs` com cinco FLACs: 24 bits/96 kHz, 16 bits/44,1 kHz, mono, um com metadados de faixa estendidos e um corrompido com quadro inválido no byte 24 912 | os cinco caminhos são passados numa única invocação em modo de lote não interativo e somente de análise, sem exportação, na ordem recebida, e o relatório por arquivo é lido no fim junto do código de saída para conferir que nenhum `fix` saiu aplicado | correção nunca é aplicada sem previsualização |
| `TASK-E18-01-AC-5` | `fixtures/tagged-16-44100.flac`, FLAC 16 bits/44,1 kHz com TITLE, ARTIST e TRACKNUMBER em `VORBIS_COMMENT`, cujo bloco reescrito fica 240 bytes maior por causa do comentário de processamento gerado | o arquivo é exportado, o `VORBIS_COMMENT` da saída é lido byte a byte com cada campo comparado ao original, e o log de sessão e o arquivo de evidência são abertos para procurar a versão do motor e os parâmetros usados | saída registra versão do motor e parâmetros usados. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-restore` | verde no CI Gate |
| Oráculo independente | Gerar as fixtures com o ffmpeg já cacheado no repo (`D:\Projects\vynamp-media-player\.cache\tools\ffmpeg\ffmpeg-n8.1.3-23-g5c4f5744cd-win64-gpl-8.1\bin\ffmpeg.exe`, fora do PATH e exposto por `AMBIENTE.ps1` em `$env:FFMPEG`), no padrão de `libs/decode/tools/fixtures/generate-flac-dev.ps1`, e rodar `cd apps/restore && cargo run --release -- --analyze --list-defects fixtures/clipping-16-44100.flac`, depois `cargo run --release -- --export /tmp/out`, com `sha256sum` da origem antes e depois, pico real por `ffmpeg -i /tmp/out/clipping-16-44100.flac -af astats -f null -`, campos por `ffprobe -show_format -show_streams` e o `VORBIS_COMMENT` lido a partir do cabeçalho de bloco. | Esperado: o relatório traz `clipping` com `severity` `severe`, `evidence` com as 4 800 amostras em 0 dBFS e as 900 perdidas por canal, até 6 `examples` e `fix` com `declip`; o `sha256` da origem é idêntico antes e depois; o export com −3 dB põe o pico do `astats` em −3,0 dBFS ± 0,05 e mantém 1 764 000 amostras; no AC-3 o destino mantém o `sha256` quando o usuário aborta (código 4) e muda quando autoriza (código 0); no lote, quatro arquivos ganham entrada no relatório e o corrompido termina com entrada de falha nomeando o byte 24 912 sem interromper os outros quatro, e o §1.3 não define o código agregado de um lote com falha parcial, então esta TASK precisa fixá-lo no contrato antes de prová-lo; no AC-2 a escrita atômica se prova pelo temporário no mesmo volume, que existe durante a escrita e some depois do rename, com o sha256 da origem idêntico antes e depois, a contagem de amostras igual e TITLE, ARTIST, TRACKNUMBER e os marcadores iguais campo a campo — a queda de 6,0 dB ± 0,5 dB na banda de repouso não é prometível hoje porque a spec §1.2 nega redução espectral (Wiener) na 1.0 e o §3.2 não tem presets de denoise, então esse número depende da cadeia que o projeto ainda vai escolher; no AC-5 o `VORBIS_COMMENT` cresce exatamente 240 bytes e o log traz `engine_version` e os parâmetros. Hoje nada disso existe: `apps/restore/src/main.rs` é `fn main() { println!("vynamp_restore 0.1.0"); }`, o `Cargo.toml` não declara dependência, não há `tests/` nem `fixtures/`, `cargo build --release` compila o binário em segundos e ele imprime a mesma linha com código 0 mesmo com `--help`, porque nem o argumento é lido; `apps/restore/specs/restore.md` §3 e §5 ainda seguem sem autoria, e o contrato de invocação do §1.3 da spec só tem `--analyze`, `--list-defects`, `--export <dir>`, `--format`, `--no-metadata`, `--overwrite`, `--version` e `--help` — `--gain-db`, `--out`, `--batch`, `--no-interaction` e `--report` ainda não existem e esta TASK tem de acrescentá-los ao contrato antes de usá-los. |
| Hardware | nenhum: a prova é de arquivo e medição numérica num único host Windows; nenhum dispositivo de áudio é aberto, o pico e o RMS saem do PCM decodificado e do `astats` lido sobre o arquivo, e a ausência de aparelho Android, iOS, macOS e de placa ASIO/CoreAudio não bloqueia nenhum dos cinco ACs. | ausência é limitação registrada, não aprovação |

## DoD — pronto para review

- [ ] Todos os critérios de aceite com evidência
- [ ] `cargo fmt --check` e `cargo clippy --all-targets -- -D warnings` limpos
- [ ] `cargo test` verde; testes ignorados listados com motivo
- [ ] Spec do módulo atualizada se a implementação revelou algo novo
- [ ] `backlog.json` com o estado correto
- [ ] Revisão feita por contexto limpo, não pelo implementador
- [ ] PR aberto com template preenchido; CI Gate verde no SHA final
## Notas de execução

Preenchido durante a execução, não antes.
