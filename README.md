# Estruturas concorrentes e lock-free

Material da aula sobre estruturas de dados concorrentes e lock-free, do Mestrado em Ciência da Computação da Universidade Federal do Ceará (UFC).

**Site:** https://jandernunes.github.io/estruturasconcorrentes/

## Conteúdo

| Arquivo | Descrição |
| --- | --- |
| [`guia-de-estudos.md`](guia-de-estudos.md) | Guia de estudos slide a slide, com questões para revisão, glossário e referências |
| [`slides/estruturas-concorrentes-slides.pdf`](slides/estruturas-concorrentes-slides.pdf) | Os 32 slides da aula |
| [`simulador/index.html`](simulador/index.html) | Simulador interativo: abra no navegador (ou pelo site acima) |
| [`assets/`](assets) | QR Code para este repositório (PNG e SVG) |

## Simulador

Uma página HTML única, sem instalação. Modos e cenários:

- **Sem sincronização:** a condição de corrida e o depósito perdido.
- **Com lock:** exclusão mútua, fila de espera e convoying ("Suspender no pior momento").
- **Lock-free (CAS):** o laço ler, calcular, CAS, repetir.
- **Lock x lock-free:** as duas versões lado a lado, com uma thread suspensa.
- **Cenários guiados:** Deadlock (e a ordem global de aquisição), Problema ABA (sem e com versão) e Fila de Michael-Scott (ajuda mútua).

Use o modo **Passo a passo**: cada clique executa uma única ação. O guia de estudos indica, slide a slide, o que observar.

## Livro-base

HERLIHY, M.; SHAVIT, N.; LUCHANGCO, V.; SPEAR, M. *The Art of Multiprocessor Programming*. 2. ed. Cambridge: Morgan Kaufmann, 2020.
