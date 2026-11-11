# vynamp-restore

**English** · [Português](#português--vynamp-restore)

Audio restoration, in its own process, with the engine's shared motor.

> **Status: specification.** This application is not implemented. The repository holds the
> module specification, its task list and the tooling both are built with. It ships no
> running code and makes no measured claim.

**Specs:** every file in [`specs/`](specs/) is at v0.2.0, an authored draft not yet approved by the maintainer. No number in them is a contract until it is approved.

Licence: `MIT OR Apache-2.0` — see [LICENSE](LICENSE), [LICENSE-MIT](LICENSE-MIT) and
[LICENSE-APACHE](LICENSE-APACHE).
Security: [GitHub Security Advisories](https://github.com/jimmykedlyn/vynamp-restore/security/advisories/new)
— see [SECURITY.md](SECURITY.md).
Contributing: [CONTRIBUTING.md](CONTRIBUTING.md).

## Scope

Restores. It does not convert format, does not play and does not edit metadata.

## Builds on its own

```bash
cargo test
```

The agent contract is the umbrella's:
[AGENTS.md](https://github.com/jimmykedlyn/vynamp/blob/develop/AGENTS.md).

## CI

GitHub Actions is the only CI (ADR-014).
[`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs `fmt`, `clippy` and `test` on
`ubuntu-22.04`, then a `build` matrix that compiles a target for Windows, Linux, macOS Intel
and macOS Apple Silicon on every pull request to `develop`, and the job `gate` publishes the
required status **`CI Gate`**. It runs with `if: always()` and reads `needs.*.result`, so a
dependency that fails, is cancelled or never reports fails the gate — there is no path to a
green gate without the real checks. Actions are pinned by full commit SHA; a floating tag is
an unreviewed dependency.

The `build` matrix is a compile gate, not a release: nothing is uploaded or signed, and it
proves only that the binary still builds on each target. Unlike the libraries, this workflow
has no `machete` job, and `[dependencies]` in `Cargo.toml` is still empty.

# Português — vynamp-restore

[English](#vynamp-restore) · **Português**

Restauração de áudio, em processo próprio, com o motor compartilhado do engine.

> **Estado: especificação.** Este aplicativo não está implementado. O repositório contém
> a especificação do módulo, a lista de tarefas dele e o tooling com que os dois são
> construídos. Não entrega código em execução e não afirma resultado medido.

**Specs:** todos os arquivos em [`specs/`](specs/) estão em v0.2.0, rascunho autorado ainda não aprovado pelo mantenedor. Nenhum número neles é contrato antes da aprovação.

Licença: `MIT OR Apache-2.0` — ver [LICENSE](LICENSE), [LICENSE-MIT](LICENSE-MIT) e
[LICENSE-APACHE](LICENSE-APACHE).
Segurança: [GitHub Security Advisories](https://github.com/jimmykedlyn/vynamp-restore/security/advisories/new)
— ver [SECURITY.md](SECURITY.md).
Contribuições: [CONTRIBUTING.md](CONTRIBUTING.md).

## Escopo

Restaura. Não converte formato, não reproduz e não edita metadados.

## Compila sozinho

```bash
cargo test
```

O contrato do agente é o do umbrella:
[AGENTS.md](https://github.com/jimmykedlyn/vynamp/blob/develop/AGENTS.md).

## CI

O GitHub Actions é a única CI (ADR-014). O
[`.github/workflows/ci.yml`](.github/workflows/ci.yml) roda `fmt`, `clippy` e `test` no
`ubuntu-22.04`, depois uma matriz `build` que compila um alvo para Windows, Linux, macOS
Intel e macOS Apple Silicon em todo pull request para `develop`, e o job `gate` publica o
status exigido **`CI Gate`**. Ele roda com `if: always()` e lê `needs.*.result`, então uma
dependência que falha, é cancelada ou nunca reporta reprova o gate — não há caminho para um
gate verde sem as verificações reais. As actions são fixadas por SHA completo de commit; uma
tag flutuante é dependência não revisada.

A matriz `build` é portão de compilação, não release: nada é enviado nem assinado, e ela
prova só que o binário ainda compila em cada alvo. Diferente das bibliotecas, este workflow
não tem job `machete`, e `[dependencies]` do `Cargo.toml` ainda está vazio.
