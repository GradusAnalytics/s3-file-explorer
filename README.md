# gradus-freeform-template

Template base para ferramentas freeform da Gradus Consultoria. Inclui o workflow de CI/CD que publica automaticamente no GitHub Pages a cada push nas branches `main` e `dev`.

---

## Como usar

### 1. Criar repositório a partir deste template

No GitHub, clique em **Use this template → Create a new repository**. Dê um nome descritivo ao repositório (ex: `OceanPact-CBO_Hub_Forca_Tarefa`).

### 2. Clonar localmente

```bash
git clone https://github.com/GradusAnalytics/<nome-do-repo>.git
cd <nome-do-repo>
```

### 3. Adicionar o HTML da ferramenta

Substitua o `index.html` ou adicione seu arquivo HTML na raiz do repositório. Use nomes descritivos e sem versão (ex: `hub-pmi.html`, `snapshot-manager.html`).

### 4. Commitar e fazer push

```bash
git add .
git commit -m "feat: adiciona ferramenta <nome>"
git push origin main
```

O workflow dispara automaticamente e publica no GitHub Pages.

### 5. Ativar o GitHub Pages

No repositório → **Settings → Pages → Branch: `gh-pages` → `/ (root)` → Save**.

### 6. Criar a branch dev

```bash
git checkout -b dev
git push origin dev
```

---

## Padrão de URLs

Após a publicação, as ferramentas ficam disponíveis nos seguintes endereços:

| Branch | URL |
|--------|-----|
| `main` | `https://gradusanalytics.github.io/<nome-do-repo>/main/<arquivo>.html` |
| `dev` | `https://gradusanalytics.github.io/<nome-do-repo>/dev/<arquivo>.html` |
| Index geral | `https://gradusanalytics.github.io/<nome-do-repo>/` |

**Exemplos:**
```
https://gradusanalytics.github.io/OceanPact-CBO_Hub_Forca_Tarefa/main/hub-pmi.html
https://gradusanalytics.github.io/OceanPact-CBO_Hub_Forca_Tarefa/dev/hub-pmi.html
```

---

## Registrar no PPR

1. Acesse o PPR → **Solicitar nova ferramenta**
2. Preencha:
   - **Modo de renderização**: `Freeform`
   - **URL do conteúdo freeform**: URL do GitHub Pages da branch `main`
   - **Manifest freeform**: `{"nome": "<nome da ferramenta>", "versao": "1.0"}`

Para ambiente de desenvolvimento, crie uma segunda ferramenta no PPR apontando para a URL da branch `dev`.

---

## Fluxo de desenvolvimento

```
dev  ──push──▶  workflow  ──▶  gh-pages/dev/   ──▶  URL dev no PPR
main ──push──▶  workflow  ──▶  gh-pages/main/  ──▶  URL prod no PPR
```

Quando a feature estiver validada na `dev`, faça merge para `main`:

```bash
git checkout main
git merge dev
git push origin main
```

---

## Estrutura do repositório

```
<nome-do-repo>/
├── .github/
│   └── workflows/
│       └── deploy-pages.yml   ← workflow de CI/CD (não editar)
├── sua-ferramenta.html        ← HTML da ferramenta
└── README.md
```

---

## Observações

- O repositório precisa ser **público** para o GitHub Pages gratuito funcionar
- O workflow copia automaticamente todos os arquivos `.html` da branch para a pasta correspondente no `gh-pages`
- Após um push, aguarde **1 a 3 minutos** para a URL refletir as alterações
- Se a URL no PPR não atualizar imediatamente, use `Ctrl+Shift+R` para forçar o reload do cache
