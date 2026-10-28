# Guia Completo de Segurança, Migração e Publicação
## PINA Ports Hub • Browser & Native PC Ports

Este documento apresenta a arquitetura, o plano de migração, os arquivos implementados e o passo a passo para configurar o **Supabase** e publicar a plataforma com **autenticação individual, aprovação de contas pelo administrador e controle estrito de sessão única server-side**.

---

## 1. Relatório de Análise e Riscos da Estrutura Anterior (Fase 1)

### Como os jogos eram carregados no GitHub Pages original
No projeto original (`https://allyson777.github.io/port-games/`), a aplicação operava como uma página puramente estática:
1. Todos os dados dos jogos (títulos, URLs diretas de execução e links para executáveis .EXE) estavam **hardcoded** dentro do arquivo `index.html` em arrays JavaScript (`defaultBrowserGames` e `defaultNativeProjects`).
2. O carregamento dos jogos era feito diretamente por tags `<a>` apontando para URLs externas (ex: `https://f12006.online/`, `https://tekken5.online/`, releases do GitHub).
3. Todo o estado do usuário (favoritos e jogos personalizados) ficava gravado apenas no `localStorage` do navegador.

### Diagnóstico de Segurança (Regras 11 e 12)
* **Arquivos e links publicamente expostos:** No GitHub Pages, **qualquer visitante** ou robô tem acesso integral ao código-fonte, aos arrays de jogos e a todas as URLs protegidas através do DevTools (F12) ou baixando o repositório.
* **Falsa segurança em clientes estáticos:** Colocar uma tela de login simples em JavaScript puro ou CSS (`display: none`) **não oferece proteção real**, pois basta inspecionar o código ou ler o array na memória para contornar qualquer barreira.
* **Solução arquitetural implementada:**
  1. Removemos a exposição pública das URLs e dados sensíveis dos jogos no cliente estático.
  2. Migramos todo o catálogo para uma tabela PostgreSQL (`public.games`) protegida por **Row Level Security (RLS)**.
  3. Os jogos e links só são entregues por meio de endpoints autenticados (`/api/games`) ou consultas RLS que validam se o usuário está logado, com status `approved` e com **sessão única ativa**.

---

## 2. Arquitetura Segura Implementada (Fases 2, 3 e 4)

```
[ Navegador do Usuário ]
          │
          │ 1. Login com Email/Senha (Supabase Auth)
          ▼
[ Supabase Auth ] ──(JWT)──► [ Backend Server / Edge Function ]
                                         │
                                         ├─ 2. Valida Status ('approved' vs 'pending')
                                         ├─ 3. Invalida Sessões Anteriores (Sessão Única)
                                         ├─ 4. Emite 'active_session_token'
                                         │
                                         ▼
                             [ PostgreSQL com RLS ]
                             - public.profiles
                             - public.games (Catálogo Protegido)
                             - public.session_logs (Auditoria)
```

### Principais Componentes:
1. **Supabase Auth:** Autenticação robusta por e-mail e senha, com fluxo de recuperação de senha e e-mails de confirmação.
2. **Cadastro Restrito:** Todo usuário que se cadastra recebe automaticamente status `pending`. Apenas administradores podem aprovar ou criar contas diretamente aprovadas.
3. **Controle de Sessão Única Server-Side (Regras 8 e 9):**
   - A cada login, o backend gera um `active_session_token` criptográfico aleatório e atualiza o registro do usuário em `public.profiles`.
   - Se o usuário tentar acessar a conta em um segundo dispositivo ou navegador, um novo token é gravado.
   - A sessão anterior é **imediatamente invalidada** tanto nas requisições de API quanto nos heartbeats contínuos (a cada 25s), retornando `403 SESSION_DISPLACED` e bloqueando a interface.
4. **Isolamento de Credenciais (Regra 7):**
   - A chave pública `SUPABASE_ANON_KEY` é usada apenas no cliente para autenticação básica.
   - A chave administrativa `SUPABASE_SERVICE_ROLE_KEY` é mantida estritamente nas variáveis de ambiente do backend ou das Supabase Edge Functions.

---

## 3. Estrutura de Arquivos do Projeto

| Arquivo / Diretório | Descrição |
|---|---|
| `index.html` | Frontend completo preservando 100% o design original, integrado ao Supabase, com telas de login, bloqueio para usuários pendentes, modal de sessão concorrente e painel admin. |
| `ps-pc-ports-hub.html` | Cópia sincronizada do frontend seguro. |
| `supabase/migrations/20261028_init_auth_and_security.sql` | Script SQL completo com tabelas, RLS, triggers e funções seguras de administração e sessão única. |
| `supabase/migrations/20261028_seed_games.sql` | Seed com todos os 73 jogos originais migrados para o banco de dados. |
| `supabase/functions/admin-action/index.ts` | Supabase Edge Function (Deno) para operações administrativas com `SERVICE_ROLE_KEY`. |
| `server/server.py` | Backend de referência em Python (FastAPI/HTTP) para validação de sessão única e proteção de recursos. |
| `server/server.js` | Backend de referência em Node.js / Express para deploy em plataformas como Vercel, Render ou Railway. |
| `server/games_catalog.json` | Catálogo estruturado de todos os jogos extraídos do projeto original. |
| `tests/test_security_and_sessions.py` | Suíte de testes automatizados de segurança e concorrência de sessão (100% aprovada). |
| `.env.example` | Modelo de variáveis de ambiente. |

---

## 4. Passo a Passo de Configuração do Supabase (Fase 7)

### Passo 1: Criar o Projeto no Supabase
1. Acesse [https://supabase.com](https://supabase.com) e crie uma conta ou faça login.
2. Clique em **"New Project"**, defina o nome (ex: `pina-ports-hub`) e crie uma senha forte para o banco de dados.
3. Aguarde o provisionamento do projeto (cerca de 2 minutos).

### Passo 2: Executar as Migrações SQL
1. No painel do seu projeto Supabase, acesse **SQL Editor** no menu lateral esquerdo.
2. Clique em **"New Query"**.
3. Copie o conteúdo de `supabase/migrations/20261028_init_auth_and_security.sql`, cole no editor e clique em **"Run"**.
4. Abra uma nova query, copie o conteúdo de `supabase/migrations/20261028_seed_games.sql` e clique em **"Run"**.
5. Verifique em **Table Editor** se as tabelas `profiles`, `games`, `user_favorites` e `session_logs` foram criadas com os 73 jogos cadastrados.

### Passo 3: Criar o Primeiro Administrador
O trigger configurado define que o **primeiro usuário cadastrado na base torna-se automaticamente Administrador com status `approved`**.
1. Cadastre seu e-mail de administrador na tela de registro ou via painel do Supabase em **Authentication > Users > Add User**.
2. Ele receberá o papel `role = 'admin'` e `status = 'approved'`.

---

## 5. Como Rodar e Testar Localmente

### Executar a Suíte de Testes Automatizados de Segurança
No terminal, execute:
```bash
python -m unittest tests/test_security_and_sessions.py
```
**Cenários validados pelos testes:**
- Visitante não autenticado tem acesso bloqueado ao catálogo (HTTP 401).
- Usuário pendente não consegue acessar os jogos (HTTP 403 `ACCOUNT_PENDING`).
- Usuário aprovado consegue acessar o catálogo completo.
- Concorrência de sessão única: login simultâneo invalida a sessão anterior com `SESSION_DISPLACED`.
- Usuário suspenso perde imediatamente autorização.
- Usuário comum não consegue executar ações administrativas nem criar administradores.

### Executar o Backend Server Localmente
```bash
# Servidor Python:
python server/server.py

# OU Servidor Node.js:
npm install
npm start
```
O servidor ficará ativo em `http://localhost:3000`.

---

## 6. Publicação e Hospedagem Compatível (Fase 7)

Para garantir que as regras de segurança funcionem sem expor credenciais secretas, recomendamos as seguintes opções de hospedagem:

### Opção A: Frontend na Vercel / Netlify + Backend no Render / Railway
1. **Frontend (`index.html`):** Publique no **Vercel**, **Netlify** ou **Cloudflare Pages**.
2. **Backend (`server/`):** Publique no **Render** (Web Service gratuito) ou **Railway**.
3. Configure as variáveis de ambiente no painel do Render/Railway:
   - `SUPABASE_URL`
   - `SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY`
4. No frontend, aponte a URL da API para o domínio gerado pelo Render/Railway.

### Opção B: Arquitetura 100% Serverless no Supabase
1. Publique o frontend no Vercel ou Netlify.
2. Implante a Edge Function com a CLI do Supabase:
   ```bash
   supabase functions deploy admin-action --no-verify-jwt
   ```
3. O Supabase cuidará de gerenciar autenticação, RLS e funções serverless sem necessidade de servidor dedicado.

