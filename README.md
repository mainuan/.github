# mainuan/.github

Repositório de configuração da organização [Mainuan](https://github.com/mainuan).

| Caminho | Função |
|---|---|
| `profile/README.md` | Página de apresentação exibida em github.com/mainuan |
| `.github/workflows/debian-package.yml` | Pipeline reutilizável para os pacotes Debian dos aplicativos |

## Pipeline reutilizável de pacotes Debian

O repositório que usa o pipeline mantém um diretório `debian/`. A versão vem
somente de `debian/changelog`, e a tag de release é `v<versão>` (com `~`
trocado por `-`, por exemplo `1.1~rc1` → `v1.1-rc1`).

```yaml
name: CI
on:
  push:
    branches: [main]
    tags: ['v[0-9]*']
  pull_request:
  workflow_dispatch:
permissions:
  contents: read
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref_type != 'tag' }}
jobs:
  deb:
    permissions:
      contents: write   # usado apenas pelo job de release, em tags
    uses: mainuan/.github/.github/workflows/debian-package.yml@<commit>
    with:
      package: nome-do-pacote
      release-title: Nome do Projeto
      test-command: python3 -m unittest discover -s tests
      smoke-command: nome-do-programa --help
```

Etapas:

1. **build** (contêiner `ubuntu:26.04` fixado por digest): dependências de
   `debian/control`, conferência da versão com a tag, testes do projeto,
   `dpkg-buildpackage`, recusa de pacote vazio, `lintian`, `SHA256SUMS` e
   notas de versão a partir de `debian/changelog`.
2. **install-test** (contêiner limpo): instalação com `apt`, conferência dos
   arquivos, teste de fumaça como usuário comum, reinstalação, remoção e
   expurgo, com falha se sobrar arquivo do pacote.
3. **release** (apenas em tag `v*` enviada ao GitHub): sem checkout, confere
   checksums e o commit da tag, cria a release e falha se ela já existir.
   Versões com `~` viram *pre-release*.

Pull requests e pushes na `main` só validam; nada é publicado.

Dependências do Mainuan que não estão no arquivo da Ubuntu (por exemplo,
`mainuan-identity` para o Welcome) entram no install-test por `extra-debs`,
uma por linha, sempre fixadas por release e SHA-256:

```yaml
      extra-debs: |
        https://github.com/mainuan/dot-files/releases/download/v2026.11.19/mainuan-identity_2026.11.19_all.deb 0b0b84a0…
```

Só URLs `https://github.com/mainuan/*/releases/download/*.deb` são aceitas.
