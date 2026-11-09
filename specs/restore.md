# Spec — vynamp-restore · restore

| Campo | Valor |
|---|---|
| Módulo | `vynamp-restore` |
| Crate(s) | a definir na autoria |
| Tipo | app |
| Versão da spec | 0.1.0 |
| Data | 2026-11-08 |
| Estado | esqueleto — §3 a §10 pendentes de autoria |
| Dono | a definir |
| Depende de | `vynamp-engine`, `vynamp-dsp` |
| É dependido por | ninguem |

> **Esqueleto gerado pelo passo 37.** §1, §2 e §7 foram decididos em
> `01-arvore-final.md` §2 e valem como norma. §3 a §6 e §8 a §10 exigem projeto de
> API e **não** foram escritos: inventá-los aqui seria fabricar a autoridade que o SDD
> exige. Estado só passa a `aprovada` com §5 preenchida.
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

## 3. API pública

PENDENTE — projeto de API. Só o que estiver aqui é contrato.

## 4. Requisitos

PENDENTE — cada requisito com ID estável `RESTORE-REQ-nn` e REQ de produto,
rastreável até uma TASK e um teste.

## 5. Critérios de aceite

PENDENTE — Given/When/Then, coerente de fora. Enquanto esta seção estiver vazia,
nenhuma TASK deste módulo pode sair de `planned`.

## 6. Erros e cancelamento

PENDENTE — erro é dado, não `panic!`. Cancelamento é explícito, não timeout.

## 7. Fora de escopo

Modelos de IA: download opt-in com SHA-256 e selecao no painel (ADR-015); nenhum modelo no instalador

## 8. Dependências

Depende de: `vynamp-engine`, `vynamp-dsp`

É dependido por: ninguem

## 9. Testes e fixtures

PENDENTE — oráculo numérico onde couber. Teste que depende de arquivo grande vive
atrás de `#[ignore]` com motivo escrito e variável de ambiente documentada.

## 10. Limitações conhecidas

PENDENTE — limitação declarada não reprova a entrega; limitação omitida reprova.
