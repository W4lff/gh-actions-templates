# gh-actions-templates

Workflow reusável de CI/CD (`devsecops-build.yml`) compartilhado por
todos os repos de aplicação do lab HashiCorp — um template só, em vez
de reimplementar a mesma esteira de build/segurança em cada repo.

## O que a esteira cobre

1. **Secret scanning** (gitleaks) — bloqueia o build se achar segredo
   em texto no código.
2. **Auditoria de dependências** (`npm audit`) — só quando `node: true`.
3. **SAST** (CodeQL, JavaScript/TypeScript) — só quando `node: true`.
4. **Testes + cobertura** (`npm test`, relatório publicado como
   artefato) — só quando `node: true`.
5. **Build da imagem** Docker.
6. **Scan de vulnerabilidades da imagem** (Trivy) — derruba o build em
   CVE crítica/alta sem correção disponível ignorada.
7. **SBOM** (Software Bill of Materials, formato SPDX) — publicado
   como artefato.
8. **Push** pro ghcr.io — só depois de passar em tudo acima.

Repositório **público** de propósito: assim qualquer repo do lab (a
maioria privada) pode chamar esse workflow sem precisar de configuração
extra de permissão entre repositórios.

## Como usar

No repo do app, `.github/workflows/deploy.yml`:

```yaml
jobs:
  build:
    uses: W4lff/gh-actions-templates/.github/workflows/devsecops-build.yml@main
    with:
      name: minha-app
      image: ghcr.io/w4lff/minha-app
      context: .
      node: true          # false se não tiver package.json
    permissions:
      contents: read
      packages: write
      security-events: write
    secrets: inherit

  deploy:
    needs: build
    runs-on: [self-hosted, hashicorp-lab]
    steps:
      - uses: actions/checkout@v4
      - name: Instalar nomad CLI
        run: |
          curl -fsSL -o nomad.zip "https://releases.hashicorp.com/nomad/2.0.4/nomad_2.0.4_linux_amd64.zip"
          unzip -o nomad.zip -d /usr/local/bin
      - name: Deploy
        run: nomad job run -detach -var="image_tag=${{ needs.build.outputs.image_tag }}" meu-job.nomad.hcl
```

Repos com mais de uma imagem (ex: `tasks-app`, que builda API e front)
chamam o template uma vez por imagem, com `name` diferente pra cada
(evita colisão de nome de artefato entre os dois builds no mesmo run).
