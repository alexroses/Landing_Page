# Documentação de Manutenção — Seu Gerente Digital

Documentação técnica do sistema para suporte e manutenção. Inclui arquitetura, configuração, banco de dados, rotas e procedimentos comuns.

---

## 1. Visão geral

**Seu Gerente Digital** é um sistema web de controle de estoque e gestão comercial (cadastros, vendas, compras, caixa, contas a pagar/receber, relatórios, mala direta e DRE). Desenvolvido em **Node.js** com **TypeScript**, **Express**, **Sequelize** (MySQL) e **Handlebars** para as views.

- **Pacote npm:** `controleestoque` (nome no `package.json`)
- **Entry point:** `src/server.ts` → inicia `src/app.js` (Express) e conecta ao banco
- **Build:** TypeScript compilado para `dist/`; views copiadas de `src/views` para `dist/views`

---

## 2. Stack tecnológica

| Camada        | Tecnologia |
|---------------|------------|
| Runtime       | Node.js (ESM) |
| Linguagem     | TypeScript 5.x |
| Servidor web  | Express 5.x |
| Banco de dados| MySQL 2 (driver mysql2) + Sequelize 6.x |
| Autenticação  | Passport (estrategia local), sessão express-session |
| Sessão (prod) | connect-session-sequelize (tabela `sessions`) |
| Templates     | express-handlebars 7.x |
| PDF           | PDFKit |
| E-mail        | Nodemailer |
| Upload        | Multer |
| Segurança     | Helmet, CORS, bcrypt para senhas |
| Build         | `node_modules/typescript/bin/tsc` via `scripts/build.mjs` |

---

## 3. Estrutura do projeto

```
SeuGerenteDigital/
├── .env                    # Variáveis de ambiente (não versionar)
├── .env.example            # Exemplo de variáveis
├── package.json
├── scripts/
│   └── build.mjs           # Build: tsc + cópia de src/views → dist/views
├── tsconfig.json
├── tsconfig.build.json     # Usado pelo build
├── public/                 # Arquivos estáticos (CSS, JS, imagens servidas pelo Express)
│   ├── css/
│   ├── js/                 # Scripts do front (ex.: produtoFormKit.js, vendaForm.js)
│   └── ...
├── src/
│   ├── server.ts           # Inicialização: banco, ensure*, listen
│   ├── app.ts              # Configuração Express (middlewares, views, rotas)
│   ├── config/
│   │   ├── database.ts      # Conexão Sequelize (MySQL)
│   │   ├── session.ts      # Sessão (memória ou banco em produção)
│   │   ├── passport.ts    # Estratégia local
│   │   ├── upload.ts      # Multer: produtos, estabelecimento, sistema, contato
│   │   ├── publicDir.ts   # Raiz de arquivos públicos (PUBLIC_WEB_DIR ou public)
│   │   └── ensure*.ts     # Migrações “soft”: colunas/tabelas/dados iniciais
│   ├── controllers/        # Lógica das rotas
│   │   ├── homeController.ts
│   │   ├── usuariosController.ts
│   │   ├── logsController.ts
│   │   ├── contatoController.ts
│   │   └── inventory/      # Caixa, produtos, compras, vendas, relatórios, etc.
│   ├── middlewares/
│   │   ├── authMiddleware.ts   # isAuthenticated, isAdministrator, isRoot
│   │   ├── flashMiddleware.ts
│   │   └── estabelecimentoLocals.ts  # Dados do estabelecimento para as views
│   ├── models/
│   │   ├── User.ts
│   │   ├── UserLog.ts
│   │   └── inventory/      # Produto, Venda, Compra, Estabelecimento, etc.
│   ├── routes/
│   │   ├── index.ts        # Montagem de todas as rotas
│   │   ├── authRoutes.ts
│   │   ├── usuariosRoutes.ts
│   │   ├── apiRoutes.ts
│   │   └── inventory/      # Por módulo (produtos, caixa, relatorios, etc.)
│   ├── services/
│   │   └── inventory/      # reportDataService, precificacaoService, estoqueService, dreService
│   ├── helpers/
│   │   └── handlebarsHelpers.js
│   ├── views/
│   │   ├── layouts/
│   │   │   └── main.handlebars
│   │   ├── partials/       # cabecalho, rodape, mensagem, contato, catalogo, hero, carrossel, sobre
│   │   ├── home.handlebars, 404.handlebars, 500.handlebars
│   │   ├── usuarios/
│   │   └── inventory/      # Uma pasta por recurso (produtos, caixa, relatorios, etc.)
│   └── scripts/
│       └── createRootUser.ts   # Cria usuário root (npm run create-root)
├── dist/                   # Gerado pelo build (não versionar)
│   ├── server.js
│   ├── app.js
│   ├── config/, controllers/, middlewares/, models/, routes/, services/, views/
│   └── ...
└── DOCUMENTACAO-MANUTENCAO.md (este arquivo)
```

---

## 4. Banco de dados

### 4.1 Conexão

- **Arquivo:** `src/config/database.ts`
- **Dialeto:** MySQL
- **Host:** `DB_HOST` (em `.env`); se for `localhost`, é forçado para `127.0.0.1` (evita IPv6 em hospedagem compartilhada)
- **Credenciais:** `DB_NAME`, `DB_USER`, `DB_PASSWORD`
- **Fuso:** `APP_TIMEZONE` (ex.: `America/Sao_Paulo`), aplicado na conexão com `SET time_zone`

### 4.2 Modelos (tabelas)

| Modelo | Tabela (inferida) | Descrição |
|--------|--------------------|-----------|
| User | users | Usuários do sistema (login, categoria) |
| UserLog | user_logs | Log de ações (exportação, etc.) |
| UnidadeMedida | unidade_medidas | Unidades de medida |
| Marca | marcas | Marcas de produtos |
| Fornecedor | fornecedores | Fornecedores |
| ClasseProduto | classe_produtos | Classes para markup (concorrência/giro/raridade) |
| Produto | produtos | Produtos (incl. kits e serviços) |
| ProdutoKitItem | produto_kit_items | Itens dos kits (insumos + qtd/preço hora) |
| Compra | compras | Compras |
| CompraItem | compra_items | Itens da compra |
| Venda | vendas | Vendas |
| VendaItem | venda_items | Itens da venda |
| VendaKitConsumo | venda_kit_consumos | Consumo de insumos de kit por item de venda |
| CaixaMovimento | caixa_movimentos | Movimentações de caixa |
| CategoriaContaPagar | categoria_conta_pagars | Categorias de contas a pagar |
| ContaRecorrente | conta_recorrentes | Contas recorrentes |
| ContaPagar | conta_pagars | Contas a pagar |
| ContaReceber | conta_recebers | Contas a receber |
| Cliente | clientes | Clientes |
| TaxaCartao | taxa_cartaos | Taxas de cartão |
| Estabelecimento | estabelecimentos | Dados do estabelecimento (nome, logo, contato, hero, carrossel) |
| Sistema | sistemas | Configuração global (ex.: logo do sistema) |
| ProdutoEstoqueInicial | produto_estoque_iniciais | Estoque inicial por produto |
| BaixaEstoque | baixa_estoques | Baixas de estoque |
| sessions | sessions | Sessões (produção, quando USE_DB_SESSIONS=1) |

Associações (belongsTo/hasMany) estão em `src/models/inventory/init.ts`.

### 4.3 “Migrações” ensure (ordem no server.ts)

Na subida, após `sequelize.sync({ alter: false })`, são executadas as funções **ensure** que criam/alteram colunas ou dados iniciais sem usar migrations formais:

1. `ensureSessionsTableColumns` — tabela de sessões
2. `ensureCanceladoColumn` — coluna cancelado em vendas/compras
3. `ensureProdutoExtraColumns` — colunas extras em produtos
4. `ensureProdutoCatalogoOferta` — apareceNoCatalogo, emOferta, precoOferta
5. `ensureProdutoPerecivelCompraItemVencimento` — perecível e vencimento em compra_item
6. `ensureProdutoServico` — ehServico em produtos
7. `ensureProdutoKitItemPrecoUnitario` — precoUnitario em produto_kit_items
8. `ensureClassesExist` — 27 classes de produto (precificacaoController)
9. `ensureContaPagarRecorrenteColumn` — contas recorrentes
10. `ensureClienteVendaColumn` — idCliente em vendas
11. `ensureVendaPrazosColumns`, `ensureCompraPrazosColumns`, `ensureContaPagarCompraColumns`
12. `ensureTaxasCartao` — tabela de taxas de cartão
13. `ensureEstabelecimento` — registro único do estabelecimento
14. `ensureEstabelecimentoLogoColumn` — logoUrl
15. `ensureEstabelecimentoInstitucionalColumns` — dados institucionais
16. `ensureEstabelecimentoHeroCarousel` — hero + carrossel
17. `ensureSistema` — registro único do sistema
18. `ensureCategoriasContaPagar` — categorias padrão (contasPagarController)

**Importante:** ao adicionar novos ensure, manter a ordem coerente (dependências entre tabelas/colunas).

---

## 5. Variáveis de ambiente

Copie `.env.example` para `.env` e ajuste. Principais:

| Variável | Obrigatório | Descrição |
|----------|-------------|-----------|
| PORT | Não | Porta HTTP (default 3000) |
| SESSION_SECRET | Sim | Chave para assinatura da sessão |
| DB_NAME | Sim | Nome do banco MySQL |
| DB_USER | Sim | Usuário MySQL |
| DB_PASSWORD | Sim | Senha MySQL |
| DB_HOST | Não | Host MySQL (default localhost → 127.0.0.1) |
| APP_TIMEZONE | Não | Fuso (ex.: America/Sao_Paulo) |
| BASE_URL / APP_URL | Não | URL base do site (e-mails, imagens em mala direta, retornos Stripe Checkout) |
| NODE_ENV | Não | `production` ativa sessão em banco e cookie secure |
| USE_DB_SESSIONS | Não | `1` para usar tabela sessions mesmo em dev |
| FORCE_INSECURE_COOKIE | Não | `1` em produção sem HTTPS para cookie não secure |
| PUBLIC_WEB_DIR | Não | Caminho absoluto do public_html (Hostinger); ver HOSTINGER-IMAGENS.md |
| ROOT_NOME, ROOT_EMAIL, ROOT_SENHA | Para create-root | Dados do usuário root |
| CONTATO_EMAIL_DESTINO | Contato | E-mail de destino do formulário de contato |
| SMTP_HOST, SMTP_PORT, SMTP_SECURE, SMTP_USER, SMTP_PASS, SMTP_FROM | E-mail | Envio (contato, mala direta, etc.) |
| STRIPE_* (secret, prices, webhook secret) | Se usar Stripe | Ver `docs/ENV_COBRANCA_STRIPE.md` |
| STRIPE_ALERTS_ENABLED, STRIPE_ALERTS_COOLDOWN_SEC | Opcional | Alertas por e-mail em eventos críticos do webhook Stripe (anti-spam) |
| INTERNAL_NOTIFY_EMAIL | Opcional | Destino de notificações internas (Stripe, cadastros, etc.) |
| MP_* (preços/dias, token webhook, etc.) | Se usar Mercado Pago | Preços de assinatura também servem de referência na home pública |

---

## 6. Execução e build

### Desenvolvimento

```bash
npm install
# Configurar .env (banco, SESSION_SECRET, etc.)
npm run dev
```

- Servidor com `tsx` e `--watch`; lê `.env` via `--env-file=.env`.
- Acesso: `http://localhost:3000` (ou a porta definida em PORT).

### Build para produção

```bash
npm run build
```

- Compila TypeScript com `tsconfig.build.json` para `dist/`.
- Copia `src/views` para `dist/views`.
- Não copia `public/`; em produção, servir `public` pelo Express (já configurado em `app.ts` a partir de `path.join(__dirname, "../public")`; no deploy, garantir que a pasta `public` exista ao lado de `dist`).

### Produção

```bash
npm start
```

- Roda `node dist/server.js`. Definir `NODE_ENV=production` e variáveis de ambiente no host.

### Criar usuário root

```bash
npm run create-root
```

- Usa `ROOT_NOME`, `ROOT_EMAIL`, `ROOT_SENHA` do `.env` e cria usuário com `categoriaUser = 0` (root). Root acessa Configurações → Sistema (logo do sistema) e não aparece na listagem de usuários.

---

## 7. Rotas principais (mapa de URLs)

Todas as rotas abaixo (exceto login e fale-conosco) exigem autenticação (`isAuthenticated`). Onde indicado, é necessário ser administrador ou root.

| Prefixo | Arquivo de rotas | Descrição |
|--------|-------------------|-----------|
| / | index | Home (hero, carrossel, **seção Sobre**, contato), fale-conosco (POST) |
| /assinatura, /assinatura/* | index (rotas em `src/routes/index.ts`) | Assinatura, Stripe/MP (conforme `BILLING_PROVIDER`), retornos e páginas GET de cancelamento |
| (sem prefixo) | authRoutes | POST /login, GET /logout |
| /api | apiRoutes | Buscas (marcas, fornecedores, unidades, produtos), taxas-cartao, contato, fale-conosco, **webhooks** (`POST /api/webhooks/stripe`, etc.) |
| /usuarios | usuariosRoutes | CRUD usuários, alterar senha (admin para CRUD) |
| /logs | logsRoutes | Listagem e exportação de logs (admin) |
| /unidades-medida | unidadesRoutes | CRUD unidades de medida |
| /marcas | marcasRoutes | CRUD marcas |
| /fornecedores | fornecedoresRoutes | CRUD fornecedores |
| /clientes | clientesRoutes | CRUD clientes |
| /produtos | produtosRoutes | CRUD produtos (incl. kit e serviço) |
| /configuracoes | precificacaoRoutes | Precificação, taxas cartão, estabelecimento, sistema (root), backup (admin) |
| /compras | comprasRoutes | Listagem e cadastro de compras |
| /caixa | caixaRoutes | Caixa, nova venda, notas, cupom, cancelamentos, abertura, aporte |
| /contas-pagar | contasPagarRoutes | Contas a pagar, recorrentes, registrar pagamento |
| /contas-receber | contasReceberRoutes | Contas a receber, registrar recebimento |
| /dre | dreRoutes | DRE |
| /estoque | estoqueRoutes | Estoque, lotes por produto, baixa (admin) |
| /dashboard | dashboardRoutes | Dashboard e dados para gráficos |
| /relatorios | relatoriosRoutes | Central de relatórios, PDF (unidades, marcas, fornecedores, clientes, produtos, compras, vendas, caixa, estoque, contas a pagar/receber), CSV/JSON |
| /mala-direta | malaDiretaRoutes | Catálogo/ofertas: página PDF, envio em lote, envio único |

Detalhes de cada rota (GET/POST) estão nos arquivos em `src/routes/` e `src/routes/inventory/`.

---

## 8. Autenticação e perfis

- **Middleware:** `src/middlewares/authMiddleware.ts`
  - `isAuthenticated`: exige usuário logado; senão redireciona para `/login` com flash.
  - `isAdministrator`: exige `categoriaUser === 2` ou `0` (admin ou root).
  - `isRoot`: exige `categoriaUser === 0`.

- **Categorias de usuário (categoriaUser):**
  - `0` = root (manutenção; acesso a Configurações → Sistema).
  - `2` = administrador (usuários, logs, exclusões, backup, taxas, etc.).
  - Outros = usuário comum (cadastros, caixa, vendas, relatórios, etc.).

- **Sessão:** em produção (ou com `USE_DB_SESSIONS=1`) a sessão é persistida na tabela `sessions`. Cookie: `httpOnly`, `sameSite: lax`, `secure` em produção (a menos que `FORCE_INSECURE_COOKIE=1`).

---

## 9. Uploads e arquivos estáticos

- **Configuração:** `src/config/upload.ts` e `src/config/publicDir.ts`.
- **Diretório base:** Se `PUBLIC_WEB_DIR` estiver definido (ex.: Hostinger), uploads vão para esse caminho (ex.: `public_html`). Caso contrário, usa a pasta `public` do projeto.
- **Subpastas de upload:**
  - `uploads/produtos` — fotos de produtos (multer, limite 5 MB; JPG, PNG, WebP).
  - `uploads/estabelecimento` — logo, hero, carrossel (até 2 MB cada).
  - `uploads/sistema` — logo do sistema (root; 2 MB).
- **Contato:** anexo do formulário de contato é mantido em memória (multer.memoryStorage), não gravado em disco.
- Em hospedagem onde o Node não serve a raiz pública, usar `PUBLIC_WEB_DIR` apontando para o `public_html` (ver `HOSTINGER-IMAGENS.md`).

---

## 10. Relatórios e exportação

- **Central:** `/relatorios` — formulário com tipo de relatório, filtros (datas, status, vencimento) e ordenação; gera link para PDF ou exportação.
- **PDF:** controllers em `src/controllers/inventory/relatorios/pdfController.ts`. Cada relatório chama `getEstabelecimentoHeader()` (nome + logo local) e `startPdf()`; o logo aparece centralizado no cabeçalho quando existe arquivo local em `logoUrl`.
- **Dados:** `src/services/inventory/reportDataService.ts` (funções por relatório; filtros por query).
- **CSV/JSON:** `src/controllers/inventory/relatorios/exportController.ts`; rotas `/:tipo/csv` e `/:tipo/json`.
- **Logs:** exportação PDF/CSV/JSON em `src/controllers/logsController.ts`.

---

## 11. Backup e restauração

- **Rotas:** GET `/configuracoes/backup` (página), GET `/configuracoes/backup/download` (download do ZIP), POST `/configuracoes/backup/restore` (restaurar a partir de um ZIP).
- **Controller:** `src/controllers/inventory/backupController.ts`. **Backup:** gera um ZIP com `dados.json` (todas as tabelas de negócio) e pasta `uploads`. **Restauração:** recebe um ZIP gerado pelo sistema, extrai `dados.json`, restaura as tabelas (TRUNCATE + INSERT) e copia a pasta `uploads` do ZIP para o diretório público. Não restaura `users`, `user_logs` nem `sessions`. Requer confirmação no formulário e permissão de administrador. Rate-limit: backup a cada 5 min, restauração a cada 10 min. Dependências: `archiver` (gerar ZIP), `adm-zip` (extrair ZIP na restauração).

---

## 12. Mala direta

- **Controller:** `src/controllers/inventory/malaDiretaController.ts`.
- **Funcionalidades:** página de pré-visualização/PDF do catálogo e ofertas; envio em lote e envio único por e-mail. Logo do estabelecimento no banner (HTML e view Handlebars) com proporção preservada (não achatado).
- **Views:** `src/views/inventory/malaDireta/index.handlebars`, `pdf.handlebars`. E-mail é montado em HTML no controller (buildEmailHtml).
- **BASE_URL/APP_URL:** usadas para montar URLs absolutas de imagens (produtos, logo) no e-mail e no PDF.

---

## 13. Produtos especiais: Kit e Serviço

- **Serviço:** produto com `ehServico = true`. Não exige marca, unidade, estoque inicial nem preço unitário no cadastro; preço informado no caixa na venda. Se for insumo de kit, no kit usa quantidade em horas e preço por hora (`precoUnitario` em `ProdutoKitItem`).
- **Kit:** produto com `ehKit = true`; itens em `ProdutoKitItem` (idProdutoInsumo, quantidade, precoUnitario para serviço). Custo do kit calculado em `precificacaoService` (inclui preço/hora para insumos serviço).
- **Front:** `public/js/produtoFormKit.js` — busca de insumos, campo “Horas” e “Preço/hora” para insumo tipo serviço; `public/js/vendaForm.js` — permite adicionar serviço sem checagem de estoque.

---

## 14. Deploy (resumo)

1. **Banco:** Criar banco MySQL e usuário; preencher `DB_*` no `.env`.
2. **Variáveis:** Definir `NODE_ENV=production`, `SESSION_SECRET`, `APP_URL`/`BASE_URL`; opcionalmente `PUBLIC_WEB_DIR` (Hostinger: ver `HOSTINGER-IMAGENS.md`).
3. **Build:** `npm run build`.
4. **Iniciar:** `npm start` (ou PM2/systemd apontando para `dist/server.js`).
5. **Sessão:** Em produção a sessão usa o banco automaticamente; garantir que `ensureSessionsTableColumns` rode na primeira subida.
6. **Root:** Executar `npm run create-root` uma vez (com ROOT_* no .env) para ter usuário de manutenção.

---

## 15. Troubleshooting

| Problema | O que verificar |
|----------|------------------|
| Imagens não aparecem no site/e-mail | `PUBLIC_WEB_DIR` apontando para o diretório público (ex.: public_html); ver HOSTINGER-IMAGENS.md. |
| Sessão não persiste após restart | Em produção, `USE_DB_SESSIONS=1` ou `NODE_ENV=production`; tabela `sessions` criada/sincronizada. |
| Erro ao conectar no MySQL | `DB_HOST` (127.0.0.1 se localhost), firewall, usuário/senha, banco criado. |
| Logo achatado no banner | View e e-mail usam `object-fit: contain` e dimensões máximas; não usar width/height fixos que deformem. |
| Campo Preço/hora do kit não aparece | Verificar se o insumo está marcado como [Serviço] e se `produtoFormKit.js` está atualizado (data-eh-servico, style do wrap). |
| Relatório PDF sem logo | `logoUrl` do estabelecimento deve ser caminho local (ex.: /uploads/estabelecimento/...); URLs externas (http) não são usadas no PDF. |
| Erro de permissão no build | Em hospedagem, o script `build.mjs` usa `node` + caminho direto para `typescript/bin/tsc` (não depende de .bin no PATH). |

---

## 16. SaaS, assinatura (Stripe), site público e datas

### 16.1 Multi-tenant e bloqueio por assinatura

- **Contexto de empresa:** `src/context/empresaContext.ts` (`runWithEmpresa`, `req.empresaId`).
- **Bloqueio quando trial/assinatura expira:** `src/middlewares/trialAssinaturaMiddleware.ts` — redireciona para `/assinatura` exceto rotas mínimas (logout, assets, troca de senha, etc.).
- **Termos com acesso suspenso:** `GET /termos` deve permanecer acessível mesmo bloqueado (lista `rotaLiberadaComTrialExpirado` no mesmo middleware).

### 16.2 Assinatura e Stripe (resumo)

- **Página do assinante:** `GET /assinatura` — `src/controllers/assinaturaController.ts`, view `src/views/assinatura/index.handlebars`.
- **Checkout / retorno:** URLs dependem de `APP_URL` ou `BASE_URL` (ver `src/services/stripeBillingService.ts`).
- **Cancelamento (fluxo dedicado):** `GET /assinatura/cancelar/stripe` e `GET /assinatura/cancelar/recorrente` — controllers em `assinaturaController.ts`, views em `src/views/assinatura/cancelar-*.handlebars`.
- **Webhooks Stripe:** `POST /api/webhooks/stripe` — `src/controllers/stripeWebhookController.ts`; logs em `stripe_webhook_logs` (`src/services/stripeWebhookLogService.ts`, ensure em `src/config/ensureStripeWebhookLogsTable.ts`). **Auditoria global (root):** `/logs/stripe-webhooks` (CSV/JSON nas mesmas rotas). **Diagnóstico por empresa:** na página `/assinatura`, bloco colapsável “eventos recentes do Stripe” (últimos registos filtrados por `empresaId`).
- **Resolução de `idEmpresa` em webhooks antigos:** se a metadata Stripe não trouxer `idEmpresa`, o handler tenta resolver pela coluna `empresas.stripeSubscriptionId`.
- **Alertas por e-mail (opcional):** `STRIPE_ALERTS_ENABLED=true`, `STRIPE_ALERTS_COOLDOWN_SEC`, `INTERNAL_NOTIFY_EMAIL` + SMTP — ver `docs/ENV_COBRANCA_STRIPE.md` e `docs/OPERACAO_COBRANCA_STRIPE.md`.

### 16.3 Site público — seção “Sobre”

- **Partial:** `src/views/partials/sobre.handlebars` (conteúdo institucional + planos).
- **Inclusão na home:** `src/views/home.handlebars` — entre `{{> carrossel}}` e `{{> contato}}` (visível com ou sem utilizador logado).
- **Dados dos planos na home:** `src/controllers/homeController.ts` expõe `planosAssinaturaPublico` (preços/dias via `getPlanosAssinatura()` em `src/services/mercadoPagoCheckoutService.ts`, mesmo quando o MP não está em uso).
- **Menu “Sobre” no cabeçalho público:** `src/views/partials/cabecalho.handlebars` — link `#sobre` quando `{{#unless user}}` e não `ocultarNavHomePublico` (fluxos de cadastro público continuam a ocultar âncoras via `estabelecimentoLocals.ts`).

### 16.4 Fuso horário (“hoje” no negócio)

- **Regra:** evitar `new Date().toISOString().slice(0, 10)` para “data de hoje” em servidor UTC (vira o dia cedo no Brasil).
- **Utilitário:** `src/utils/businessDate.ts` — `getTodayStringInBusinessTz()`, `formatDateToYmdInBusinessTz()`, `getBusinessDayBounds()`.
- **Exemplos de uso:** `src/services/lembretesDiaService.ts`, `src/controllers/inventory/estoqueController.ts`, `src/controllers/inventory/contasPagarController.ts`.

---

## 17. Contato e referências

- **Autor (package.json):** Alexandre Rosas.
- **Documento de imagens na Hostinger:** `HOSTINGER-IMAGENS.md`.
- **Exemplo de ambiente:** `.env.example`.
- **Fases SaaS e roadmap:** `docs/SAAS_FASES.md`.
- **Relatório de funcionalidades (apresentação):** `docs/RELATORIO_FUNCIONALIDADES_E_VALOR_DE_MERCADO.md`.
- **Operação Stripe / checklist:** `docs/OPERACAO_COBRANCA_STRIPE.md`.
- **Variáveis Stripe (equipa):** `docs/ENV_COBRANCA_STRIPE.md`.

Para alterações em regras de negócio (venda, compra, caixa, precificação), os pontos principais estão nos controllers em `src/controllers/inventory/` e nos services em `src/services/inventory/` (precificacaoService, estoqueService, dreService, reportDataService).
