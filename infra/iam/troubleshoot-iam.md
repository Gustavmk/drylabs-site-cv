# Troubleshooting — OIDC AssumeRoleWithWebIdentity

Registro dos comandos usados para diagnosticar e corrigir o erro do GitHub Actions:

```
Could not assume role with OIDC: The web identity token provided could not be validated.
```

## Diagnóstico

### 1. Verificar o OIDC Provider (existência, audience e thumbprint)

```bash
aws iam list-open-id-connect-providers
```

```bash
aws iam get-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::#{ACCOUNT_ID}#:oidc-provider/token.actions.githubusercontent.com
```

Confirme dois pontos no retorno:

- `ClientIDList` deve conter exatamente `sts.amazonaws.com` (audience). Um valor com typo, como `sts.amazonaws.co` (sem o "m"), faz a AWS rejeitar o token antes mesmo de avaliar a trust policy — é exatamente essa a causa do erro *"web identity token could not be validated"*.
- `Url` deve ser `token.actions.githubusercontent.com`.

### 2. Verificar a trust policy da role

```bash
aws iam get-role \
  --role-name github-openidconnect-drylabs-site-cv \
  --query 'Role.AssumeRolePolicyDocument'
```

O `sub` na condition precisa seguir o formato `repo:<OWNER>/<REPO>:<qualifier>`, sem sufixos extras. Um erro comum é incluir IDs numéricos (ex: `Gustavmk@35900561/drylabs-site-cv@1380760089`) ou o nome do repositório errado (ex: `drylabs-site-csv` em vez de `drylabs-site-cv`) — isso faz a condition nunca dar match e a AssumeRole é negada (`AccessDenied`, erro diferente do de validação do token).

Exemplos de `sub` válidos:

```
repo:Gustavmk/drylabs-site-cv:*
repo:Gustavmk/drylabs-site-cv:ref:refs/heads/main
repo:Gustavmk/drylabs-site-cv:environment:production
```

## Correção

### 3. Corrigir a audience do OIDC Provider

Não existe update direto para `ClientIDList`; adicione o valor certo e remova o errado:

```bash
aws iam add-client-id-to-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::#{ACCOUNT_ID}#:oidc-provider/token.actions.githubusercontent.com \
  --client-id sts.amazonaws.com

aws iam remove-client-id-from-open-id-connect-provider \
  --open-id-connect-provider-arn arn:aws:iam::#{ACCOUNT_ID}#:oidc-provider/token.actions.githubusercontent.com \
  --client-id sts.amazonaws.co
```

### 4. Corrigir o `sub` na trust policy

Depois de corrigir `infra/iam/trust-policy.json` localmente, reaplique:

```bash
aws iam update-assume-role-policy \
  --role-name github-openidconnect-drylabs-site-cv \
  --policy-document file://infra/iam/trust-policy.json
```

### 5. Validar

Reexecute o workflow (`workflow_dispatch` ou novo push em `main`) e confirme que o step *Configure AWS credentials (OIDC)* passa da etapa `AssumeRoleWithWebIdentity`.

## Resumo dos erros encontrados neste projeto

| Sintoma | Causa | Arquivo/Recurso |
|---|---|---|
| `web identity token could not be validated` | `ClientIDList` do OIDC provider com typo (`sts.amazonaws.co`) | OIDC Provider na conta AWS |
| (seguinte, após corrigir a audience) `AccessDenied` esperado se não corrigido | `sub` da trust policy com sufixos `@ID` inválidos e nome de repo incorreto (`drylabs-site-csv`) | `infra/iam/trust-policy.json` |
