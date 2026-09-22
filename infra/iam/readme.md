# IAM — Role de Deploy (GitHub Actions + OIDC)

Este diretório contém a configuração do IAM que permite ao GitHub Actions fazer deploy do site no S3 e invalidar o cache do CloudFront, sem usar chaves de acesso estáticas.

## Arquivos

| Arquivo | Descrição |
|---|---|
| `trust-policy.json` | Trust policy que permite ao GitHub OIDC assumir a role, restrita ao repositório e à branch `main`. |
| `role.json` | Policy de permissões (escopo mínimo) usada pela role de deploy. |

## Permissões concedidas (`role.json`)

- `s3:ListBucket` no bucket `s3-drylabs-website` — necessário para `aws s3 sync`.
- `s3:PutObject` e `s3:DeleteObject` nos objetos do bucket — enviar e remover arquivos do site.
- `cloudfront:CreateInvalidation` na distribuição — limpar o cache após o deploy.

## Placeholders

Antes de aplicar, substitua nos arquivos:

- `ACCOUNT_ID` — ID da conta AWS (12 dígitos).
- `DISTRIBUTION_ID` — ID da distribuição CloudFront (ex: `E1A2B3C4D5E6F7`).
- `OWNER/REPOSITORY` — repositório no GitHub (ex: `github_account/aws-drylabs-site`).

## Pré-requisito: OIDC Provider

O provedor OIDC do GitHub precisa existir na conta (criado apenas uma vez por conta):

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

Verifique se já existe antes de criar:

```bash
aws iam list-open-id-connect-providers
```

## Aplicando a configuração

### 1. Criar a role com a trust policy

```bash
aws iam create-role \
  --role-name github-actions-deploy-site \
  --assume-role-policy-document file://infra/iam/trust-policy.json
```

### 2. Anexar a policy de permissões à role

```bash
aws iam put-role-policy \
  --role-name github-actions-deploy-site \
  --policy-name deploy-site \
  --policy-document file://infra/iam/role.json
```

### 3. Configurar no GitHub

Adicione o ARN da role como secret no repositório:

```
AWS_ROLE_ARN=arn:aws:iam::ACCOUNT_ID:role/github-actions-deploy-site
```

### 4. Usar no workflow (exemplo)

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
      aws-region: us-east-1

  - run: aws s3 sync src/ s3://s3-drylabs-website --delete

  - run: |
      aws cloudfront create-invalidation \
        --distribution-id DISTRIBUTION_ID \
        --paths "/*"
```

## Atualizando as policies

Após editar os arquivos JSON, reaplique com:

```bash
aws iam update-assume-role-policy \
  --role-name github-actions-deploy-site \
  --policy-document file://infra/iam/trust-policy.json

aws iam put-role-policy \
  --role-name github-actions-deploy-site \
  --policy-name deploy-site \
  --policy-document file://infra/iam/role.json
```

## Observações

- A bucket policy pública em `infra/s3/policy.json` (acesso de leitura `s3:GetObject`) é independente desta role — não deve ser atribuída a ela.
- Não use `AdministratorAccess` ou `AmazonS3FullAccess` para este fluxo.
