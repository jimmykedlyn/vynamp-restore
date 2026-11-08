# TASK-E18-01 — Entregar Vynamp Restore como ferramenta desktop de restauração

> Gerado pelo passo 37 a partir de `tasks/backlog.json`. Os campos de estado vêm do
> backlog e não devem ser editados aqui. Critério de aceite: **PENDENTE** de autoria —
> a coluna `Then` é o requisito aprovado, copiado literal; `Given` e `When` ainda
> não existem.

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

<!-- PENDENTE: o que esta TASK não faz, mesmo parecendo relacionado. -->

## Antes de começar

- [ ] Spec do módulo aprovada e o critério de aceite citado já existe nela
- [ ] Worktree própria criada (`cargo xtask repo worktree task-e18-01-claude-<slug>`)
- [ ] Fixtures/hardware necessários disponíveis ou limitação registrada
- [ ] Nenhuma TASK em execução tocando os mesmos arquivos

## Critérios de aceite

| ID | Given | When | Then |
|---|---|---|---|
| `TASK-E18-01-AC-1` | PENDENTE | PENDENTE | A fonte nunca é sobrescrita sem confirmação |
| `TASK-E18-01-AC-2` | PENDENTE | PENDENTE | escrita atômica com validação pós-escrita preserva origem e metadados |
| `TASK-E18-01-AC-3` | PENDENTE | PENDENTE | defeitos reportados têm severidade e evidência |
| `TASK-E18-01-AC-4` | PENDENTE | PENDENTE | correção nunca é aplicada sem previsualização |
| `TASK-E18-01-AC-5` | PENDENTE | PENDENTE | saída registra versão do motor e parâmetros usados. |

## Verificação

| Item | Comando / fixture | Evidência esperada |
|---|---|---|
| Teste automatizado | `cargo test` no repo `vynamp-restore` | verde no CI Gate |
| Oráculo independente | PENDENTE | comparação numérica, não "parece certo" |
| Hardware | PENDENTE | ausência é limitação registrada, não aprovação |

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
