# aws-drylabs-site

Site pessoal estático (portfólio) hospedado na AWS.

## Estrutura

```
src/            Código-fonte do site (HTML, CSS, imagens)
infra/s3/       Configuração de infraestrutura (bucket policy do S3)
```

## Infraestrutura

- **S3** — hospedagem dos arquivos estáticos do site.
- **CloudFront** — CDN para distribuição de conteúdo com baixa latência.
- **SSL** — certificado TLS/SSL via AWS Certificate Manager (ACM) para HTTPS.
- **Route53** — gerenciamento de DNS do domínio.

## Deploy

1. Enviar o conteúdo de `src/` para o bucket S3.
2. Aplicar a bucket policy em `infra/s3/policy.json`.
3. Invalidar o cache do CloudFront após atualizações.
