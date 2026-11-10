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

- **Não** implementa o motor de restauração de verdade: filtro, de-noise, declipping e ganho são `TASK-E18-02`, que ainda está `planned` com o template do passo 37 e sem critério escrito. Esta TASK entrega a ferramenta desktop que conduz o motor, os diagnósticos e a exportação; reimplementar o DSP aqui daria dois donos do mesmo algoritmo.
- **Não** faz decoding nem encoding de FLAC nem de outro container: leitura e escrita são `TASK-E01-07` (codecs modernos) e `TASK-E08-01` (encode), e a ferramenta consome essas portas. Um reader próprio aqui reintroduziria as regras de gapless e de delay que já têm dono.
- **Não** integra com o SO nem com menus de shell: abertura externa por MIME e extensão é `TASK-E16-04`, o registro de app único e o file type association são `TASK-E16-03`, e os UTTypes e document types de macOS são `TASK-E16-05`. A ferramenta é um executável de linha de comando e não reivindica nenhuma dessas superfícies.
- **Não** define a matriz de ativação nem a ownership de instalação: o owner único entre instalado e portable é `TASK-E16-08`. Executar em lote a partir de um diretório não exige nem cria lock de perfil.
- **Não** entrega a UI de desktop nem o instalador do restore: a distribuição assinada e o pacote por plataforma são de `TASK-E15-06`, `TASK-E15-07` e `TASK-E15-08`. Esta TASK entrega o binário e o seu critério; o pacote tem dono próprio.

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e18-01-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E18-01-AC-1` | um FLAC de 16 bits/44,1 kHz com 40 s, amostras limitadas em 3 trechos (clipping digital duro, pico em 0 dBFS em 4 800 amostras somadas), DR medido de 6 dB e o aviso de que 900 amostras foram perdidas acima do teto nos dois canais | o `vynamp-restore` declara o diagnóstico (clipping, pico plano e limite de bits), recebe a escolha explícita do usuário pelo ganho negativo de −3 dB, e exporta um FLAC novo ao lado do original | A fonte nunca é sobrescrita sem confirmação |
| `TASK-E18-01-AC-2` | um FLAC de 16 bits/44,1 kHz com ruído de fita injetado artificialmente (soma de um ruído com espectro 1/f e nível −42 dBFS) e os marcadores de início e fim de cada faixa que o arquivo de origem tinha | o processo de restauração roda com a opção de redução de ruído em 6 dB declarada, e o arquivo exportado é comparado com o original em dois eixos: o ruído medido na banda de repouso e a preservação dos marcadores de faixa e metadados | escrita atômica com validação pós-escrita preserva origem e metadados |
| `TASK-E18-01-AC-3` | um FLAC de 16 bits/44,1 kHz que declara 1 176 000 amostras de áudio, um encoder de referência que gera o mesmo conteúdo e um diretório de saída onde o arquivo de destino já existe com conteúdo diferente | a restauração é executada com a saída apontando para o arquivo existente, e a decisão (sobrescrever ou abortar) é tomada pelo usuário antes da escrita, com o `sha256` do destino conferido antes e depois do comando | defeitos reportados têm severidade e evidência |
| `TASK-E18-01-AC-4` | um diretório com cinco FLACs de fontes distintas (um de 24 bits/96 kHz, um de 16 bits/44,1 kHz, um mono, um com metadados de faixa estendidos e um corrompido com quadro inválido na posição de bytes 24 912) | a ferramenta recebe o diretório inteiro e é executada sem interação, com o modo de lote explícito no comando e o relatório de cada arquivo escrito no fim da execução | correção nunca é aplicada sem previsualização |
| `TASK-E18-01-AC-5` | um FLAC de 16 bits/44,1 kHz com metadados de faixa (o título, o artista e o número de faixa) que não cabem no mesmo bloco VORBIS_COMMENT depois da restauração, porque o comentário de processamento gerado é 240 bytes maior | o arquivo é restaurado e o bloco de metadados reescrito é lido diretamente, com o conteúdo de cada campo comparado contra o original e o tamanho do bloco conferido em bytes | saída registra versão do motor e parâmetros usados. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-restore` | verde no CI Gate |
| Oráculo independente | `cd apps/restore && cargo run --release -- --diagnose fixtures/clipping-16-44100.flac` e depois `--restore fixtures/clipping-16-44100.flac --gain-db -3 --out /tmp/out.flac`, com um segundo comando `ffmpeg -i /tmp/out.flac -af astats -f null -` para o pico real e `sha256sum` do destino antes e depois; para o lote, `--batch /tmp/cinco-flacs --no-interaction --report /tmp/report.json` no diretório de cinco arquivos, um deles com quadro inválido em 24 912 bytes; e para metadados, `ffprobe -show_format -show_streams` sobre a saída | Esperado: o diagnóstico nomeia clipping em 4 800 amostras, pico em 0 dBFS e perda acima do teto em 900 amostras por canal, e não escreve nada antes da escolha do usuário; o export com −3 dB leva o pico real a −3 dBFS pela medição do `astats` e o arquivo de saída tem o mesmo número de amostras e o `sha256` do destino muda só quando o usuário autoriza a sobrescrita; no lote, os quatro arquivos íntegros produzem entrada no relatório e o corrompido termina com erro nomeando o byte 24 912 e código de saída diferente de zero, sem abortar os outros; na redução de ruído, a banda de repouso cai o valor declarado e os marcadores de faixa continuam os mesmos. Hoje `apps/restore` tem 1 arquivo `.rs`, nenhum binário de restauração, nenhuma fixture em `tests/` e `specs/restore.md` §5 ainda `PENDENTE`, então nenhum desses comandos existe: a evidência é a que a TASK precisa produzir, não a que existe. |
| Hardware | nenhum: a prova é de arquivo e de medição numérica num único host Windows; nenhum dispositivo de áudio é aberto e o `astats` lê o arquivo, não a placa | ausência é limitação registrada, não aprovação |

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
