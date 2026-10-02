# squad-harness-research

Preprint e material ilustrativo sanitizado do padrão **squad-harness**: o substrato de
verificação por baixo da organização de "empresa de agentes". Tese estreita de propósito: a
camada organizacional (papéis, coordenadores, personas) é commodity, e fazer do **"não medido"
um resultado de primeira classe** é a decisão de projeto mais valiosa do sistema.

Não é aplicação, não usa porta, não tem `package.json`. Markdown, SVG e um `index.html`.

## O que é publicado, e por isso não se mexe sem cuidado

| Item | Valor |
|---|---|
| DOI | [10.5281/zenodo.22481935](https://doi.org/10.5281/zenodo.22481935) |
| Licença | CC BY 4.0 (não é a licença de código do resto da pasta) |
| ORCID | 0009-0006-7522-326X |
| Página | https://tedfernandes.github.io/squad-research/ |
| Remote | `tedfernandes/squad-research` |

O remote fica na **conta pessoal**, e isso é exceção consciente à regra de `dev/CLAUDE.md`
("a conta pessoal não guarda projeto"). A regra existe para projeto de cliente não morar no
perfil; aqui o repositório é uma publicação acadêmica assinada, com DOI e ORCID do autor, e o
DOI do Zenodo e o GitHub Pages apontam para este caminho. Transferir para uma org quebraria os
dois, e o ganho seria zero.

Versão sai por **release** no GitHub (o selo de versão do README lê a última). Mudança de
conteúdo do paper depois de publicada a versão exige nova release, porque o DOI versiona.

## Bilíngue, e os dois lados andam juntos

`paper.md` + `paper.en.md`, `README.md` + `README.en.md`. Mesma regra do `squad-harness`:
mexeu num, mexeu no outro, no mesmo commit. Selo, figura e número citado existem nas duas
versões e divergem em silêncio quando só uma é editada.

As figuras em `figures/` têm variante clara e escura (`*-dark.*`), servidas por `<picture>` com
`prefers-color-scheme`. Trocar uma figura é trocar as duas.

## Número no paper é número medido, não estimativa

O corpo cita contagens do sistema real (projetos sem teste, invariantes da catraca, cobertura).
São medições, e o paper é um relato de experiência: número que não dá para reproduzir com um
comando não entra. Ao atualizar, meça de novo em vez de ajustar o texto pela memória, e diga a
data da medição.

## Não confundir com o paper do sandbox

`sandbox/2026-09-verification-first-agents` é **outro repositório**
(`tedfernandes/business-agents-research`), com o mesmo título e estrutura parecida, parado desde
10/09/2026. Este aqui é o que seguiu: preprint PT e EN de 26/09, com DOI. Editar o do sandbox
pensando que é este é o erro fácil de cometer, porque o README dos dois abre igual.
