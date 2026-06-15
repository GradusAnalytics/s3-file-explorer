# S3 File Explorer — Gradus Consultoria

Ferramenta interna de exploração e gestão de arquivos no Amazon S3. Interface web single-page (SPA) publicada via GitHub Pages e integrada ao PPR (Portal de Produtos e Recursos) da Gradus.

---

## Visão Geral

O S3 File Explorer permite que usuários autenticados no PPR naveguem, façam upload, download, criem pastas, renomeiem e excluam arquivos no bucket S3 `diretorio-consultoria`, com controle de acesso por usuário via um sistema de permissões próprio.

A aplicação é um único arquivo HTML (`s3-explorer.html`) sem dependências externas. Toda a lógica de negócio reside em funções AWS Lambda invocadas via API Gateway REST.

---

## Funcionalidades

### Navegação
- Navegação em árvore de pastas com breadcrumb
- Painel lateral com estrutura de diretórios (sidebar)
- Indicador de conexão e status (conectado / erro)
- Contador de itens na pasta atual

### Upload
- Upload via botão ou drag-and-drop na área de listagem
- Suporte a múltiplos arquivos simultâneos
- Barra de progresso individual por arquivo
- Upload direto ao S3 via presigned URL (sem trafegar pelo backend)
- Registro automático de ownership do arquivo no sistema de compartilhamento

### Download
- Download via presigned URL gerada pelo backend
- Verificação de permissão antes de gerar a URL
- Nome do arquivo preservado no download

### Gerenciamento de Pastas e Arquivos
- Criar nova pasta (com verificação de duplicata)
- Renomear arquivos e pastas (operação recursiva para pastas)
- Excluir arquivos e pastas (exclusão recursiva para pastas, até 1.000 objetos)
- Seleção múltipla de itens para operações em lote

### Compartilhamento
- Compartilhar arquivos ou pastas com outros usuários por e-mail
- Modos de permissão: **Visualização** (`view`) e **Edição** (`edit`)
- Gerenciar compartilhamentos existentes (alterar modo, remover acesso)
- Apenas o dono (`owner`) pode compartilhar ou alterar permissões

### Sistema de Permissões
Três níveis de acesso por recurso:

| Nível | Pode visualizar | Pode editar / renomear / excluir | Pode compartilhar |
|-------|:-:|:-:|:-:|
| `owner` | ✓ | ✓ | ✓ |
| `edit` | ✓ | ✓ | ✗ |
| `view` | ✓ | ✗ | ✗ |

Recursos sem registro no sistema de compartilhamento são tratados como de propriedade do usuário logado.

### Integração PPR
- Autenticação via `postMessage` com o PPR (`ppr.init` / `ppr.ready`)
- E-mail e nome do usuário obtidos automaticamente do contexto PPR
- Modo standalone (fora do PPR): navegação sem filtro de permissão

---

## Arquitetura

```
┌─────────────────────────────────────────────┐
│  PPR (iframe host)                          │
│  ┌───────────────────────────────────────┐  │
│  │  s3-explorer.html                     │  │
│  │  (GitHub Pages — branch main)         │  │
│  │                                       │  │
│  │  fetch() ──► API Gateway ──► Lambda   │  │
│  │  XHR PUT ──► S3 Presigned URL         │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘

AWS (us-east-1)
├── S3 Bucket: diretorio-consultoria
│   └── _system/sharing.json  ← controle de permissões
└── Lambda Functions (7)
    ├── s3fe-list
    ├── s3fe-upload-url
    ├── s3fe-download-url
    ├── s3fe-mkdir
    ├── s3fe-rename
    ├── s3fe-delete
    └── s3fe-sharing
```

---

## Funções Lambda

Todas as funções:
- Runtime: **Python 3.12**
- Autenticação via header `X-Api-Token`
- CORS liberado para todas as origens (`*`)
- Erros internos logados no CloudWatch (stack trace não exposto ao cliente)

### `s3fe-list`
**Timeout:** 15s | **Método:** `GET /files`

Lista pastas e arquivos de um prefixo S3.

| Query Param | Tipo | Descrição |
|-------------|------|-----------|
| `prefix` | string | Prefixo relativo a listar (ex: `projetos/`) |
| `email` | string | E-mail do usuário logado. Se ausente: modo standalone (sem filtro) |

**Comportamento:**
- Oculta o prefixo `_system/` (pasta interna)
- Com `email`: retorna apenas recursos onde o usuário é `owner`; recursos sem registro no `sharing.json` são tratados como do próprio usuário
- Retorna `perm` por item com `mode` e `is_owner`

---

### `s3fe-upload-url`
**Timeout:** 15s | **Método:** `POST /files/upload-url`

Gera uma presigned URL para upload direto ao S3 (`PUT`). O frontend faz o upload diretamente ao S3 via `XMLHttpRequest`, sem passar pelo backend.

| Body Param | Tipo | Descrição |
|------------|------|-----------|
| `key` | string | Caminho relativo do arquivo (ex: `projetos/relatorio.pdf`) |
| `owner` | string | E-mail do usuário. Registrado como dono no `sharing.json` |
| `content_type` | string | (opcional) MIME type. Detectado pela extensão se ausente |

**Comportamento:**
- URL expira em 300s (configurável via `UPLOAD_URL_TTL`)
- Registra ownership no `sharing.json` automaticamente (não bloqueia o upload se falhar)
- Se `owner` ausente e `DEFAULT_OWNER` não configurado: não registra ownership

---

### `s3fe-download-url`
**Timeout:** 15s | **Método:** `GET /files/download-url`

Gera uma presigned URL para download (`GET`) de um objeto S3.

| Query Param | Tipo | Descrição |
|-------------|------|-----------|
| `key` | string | Caminho relativo do arquivo |
| `email` | string | E-mail do usuário. Usado para verificar permissão de leitura |

**Comportamento:**
- Verifica permissão (`view` ou superior) antes de gerar a URL
- URL expira em 600s / 10min (configurável via `DOWNLOAD_URL_TTL`)
- Retorna 403 se usuário não tiver acesso ao recurso
- `Content-Disposition: attachment` com nome do arquivo preservado

---

### `s3fe-mkdir`
**Timeout:** 15s | **Método:** `POST /files/mkdir`

Cria uma pasta (objeto vazio com sufixo `/`) no S3.

| Body Param | Tipo | Descrição |
|------------|------|-----------|
| `prefix` | string | Caminho relativo da pasta (ex: `projetos/novo/`) |
| `owner` | string | E-mail do usuário. Registrado como dono no `sharing.json` |

**Comportamento:**
- Retorna 409 se a pasta já existir
- Registra ownership no `sharing.json` se `owner` presente

---

### `s3fe-rename`
**Timeout:** 30s | **Método:** `POST /files/rename`

Renomeia (move) um arquivo ou pasta.

| Body Param | Tipo | Descrição |
|------------|------|-----------|
| `source` | string | Caminho relativo de origem |
| `destination` | string | Caminho relativo de destino |
| `email` | string | E-mail do usuário. Verificado contra permissão de edição |

**Comportamento:**
- Verifica permissão `edit` ou `owner` no recurso de origem
- **Arquivo:** copy + delete
- **Pasta:** copia recursivamente todos os objetos, depois deleta em lote (1.000/batch)
- Atualiza automaticamente o `sharing.json` (renomeia a chave do recurso)

---

### `s3fe-delete`
**Timeout:** 30s | **Método:** `DELETE /files`

Exclui um arquivo ou pasta (com todo o conteúdo).

| Body Param | Tipo | Descrição |
|------------|------|-----------|
| `key` | string | Caminho relativo do arquivo (exclusão de arquivo único) |
| `prefix` | string | Caminho relativo da pasta (exclusão recursiva) |
| `email` | string | E-mail do usuário. Verificado contra permissão de edição |

**Comportamento:**
- Verifica permissão `edit` ou `owner` antes de excluir
- Limite de 1.000 objetos por pasta (configurável via `MAX_DELETE`)
- Remove automaticamente as entradas correspondentes do `sharing.json`
- Retorna 404 se o recurso não existir

---

### `s3fe-sharing`
**Timeout:** 15s | **Métodos:** `GET`, `POST`, `DELETE`

CRUD completo do sistema de compartilhamento. Todas as operações de escrita verificam ownership antes de executar.

#### `GET /sharing?email=<email>`
Retorna todas as entradas do `sharing.json` onde o usuário é `owner` ou `recipient`. Requer `email`.

#### `GET /sharing?action=visible&email=<email>`
Retorna mapa de recursos visíveis para o usuário (próprios + compartilhados com ele).

#### `POST /sharing` — `action=register`
Registra ownership de um novo recurso.

| Body | Tipo | Descrição |
|------|------|-----------|
| `resource` | string | Caminho do recurso |
| `owner` | string | E-mail do dono |
| `type` | string | `file` ou `folder` |

#### `POST /sharing` — `action=share`
Adiciona ou atualiza um recipient em um recurso.

| Body | Tipo | Descrição |
|------|------|-----------|
| `resource` | string | Caminho do recurso |
| `owner` | string | E-mail do dono (obrigatório para validação) |
| `email` | string | E-mail do recipient |
| `mode` | string | `view` ou `edit` |

> Apenas o `owner` registrado pode executar esta ação. Retorna 403 se `owner` não bater.

#### `DELETE /sharing` — `action=unshare`
Remove um recipient de um recurso. Requer `owner` válido.

#### `DELETE /sharing` — `action=unregister`
Remove o registro completo de um recurso do `sharing.json`. Requer `owner` válido.

#### `POST /sharing` — `action=seed_defaults`
Registra `DEFAULT_OWNER` como dono de todos os recursos ainda sem registro. Requer `DEFAULT_OWNER` configurado via env var.

---

## Variáveis de Ambiente (Lambda)

| Variável | Funções | Descrição |
|----------|---------|-----------|
| `S3_BUCKET` | todas | Nome do bucket S3 |
| `ROOT_PREFIX` | todas | Prefixo raiz dentro do bucket (opcional) |
| `API_TOKEN` | todas | Token de autenticação (`X-Api-Token`) |
| `DEFAULT_OWNER` | `s3fe-mkdir`, `s3fe-upload-url`, `s3fe-sharing` | E-mail padrão de ownership quando não fornecido |
| `MAX_DELETE` | `s3fe-delete` | Limite de objetos para exclusão recursiva (padrão: 1000) |
| `UPLOAD_URL_TTL` | `s3fe-upload-url` | Validade da presigned URL de upload em segundos (padrão: 300) |
| `DOWNLOAD_URL_TTL` | `s3fe-download-url` | Validade da presigned URL de download em segundos (padrão: 600) |

---

## Deploy

### Frontend (HTML)

O arquivo `s3-explorer.html` é publicado automaticamente via GitHub Actions no GitHub Pages a cada push na branch `main`.

URL de produção:
```
https://gradusanalytics.github.io/s3-file-explorer/main/s3-explorer.html
```

### Lambdas

Os arquivos fonte estão em `aws_lambdas/`. O handler de todas as funções é `lambda_function.lambda_handler` — o arquivo deve ser zipado com esse nome.

Script de deploy (PowerShell, executar dentro de `aws_lambdas/`):

```powershell
$FUNCTION = "s3fe-NOME"
Copy-Item "$FUNCTION.py" "lambda_function.py"
Compress-Archive -Force -Path "lambda_function.py" -DestinationPath "$FUNCTION.zip"
aws lambda update-function-code --function-name $FUNCTION --zip-file "fileb://$FUNCTION.zip"
Remove-Item "$FUNCTION.zip"; Remove-Item "lambda_function.py"
```

Substituir `s3fe-NOME` pelo nome real da função (ex: `s3fe-list`, `s3fe-delete`, etc.).

---

## Estrutura do Repositório

```
s3-file-explorer/
├── .github/
│   └── workflows/
│       └── deploy-pages.yml      ← CI/CD para GitHub Pages
├── aws_lambdas/                  ← código-fonte das Lambdas (não publicado no Pages)
│   ├── s3fe-list.py
│   ├── s3fe-upload-url.py
│   ├── s3fe-download-url.py
│   ├── s3fe-mkdir.py
│   ├── s3fe-rename.py
│   ├── s3fe-delete.py
│   └── s3fe-sharing.py
├── s3-explorer.html              ← aplicação completa (SPA)
└── README.md
```

---

## Registrar no PPR

1. Acesse o PPR → **Solicitar nova ferramenta**
2. Preencha:
   - **Modo de renderização:** `Freeform`
   - **URL do conteúdo freeform:** URL do GitHub Pages (branch `main`)
   - **Manifest freeform:** `{"nome": "S3 File Explorer", "versao": "1.0"}`

A ferramenta lê o contexto do usuário (`email`, `nome`) automaticamente via `postMessage` ao carregar dentro do PPR.

---

## Observações de Segurança

- O token de API (`API_TOKEN`) deve ser configurado via variável de ambiente na Lambda, nunca hardcoded
- As permissões são verificadas **no backend** — manipulação do frontend não concede acesso
- O `sharing.json` é um arquivo S3 compartilhado entre todas as Lambdas; operações concorrentes intensas podem causar race condition (mitigação recomendada: migrar para DynamoDB)
- Uploads vão diretamente ao S3 via presigned URL — o backend não trafega o conteúdo dos arquivos
- Erros internos são logados no CloudWatch e não expostos ao cliente
