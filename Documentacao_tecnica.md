# VitrineIA — Documentação Técnica

> Plataforma de inteligência comercial para lojistas de shopping center. Visitantes respondem pesquisas anônimas em troca de pontos; os lojistas recebem segmentos de público e oportunidades de venda cruzada geradas por regras de associação e, opcionalmente, IA generativa.
>
> **Status:** protótipo funcional (ambiente de desenvolvimento). Documento atualizado em 14/11/2026.

---

## 1. Visão geral

O sistema tem dois públicos com mecanismos de identidade diferentes:

| Público | Como se identifica | O que faz |
|---|---|---|
| **Lojista** (estabelecimento) | E-mail e senha → JWT (`Authorization: Bearer`) | Cadastra vendas, cupons, campanhas e questionários; vê painel, segmentos e oportunidades; valida resgates no balcão |
| **Visitante** (consumidor) | **Sem login.** Token aleatório anônimo (`X-Visitante-Token`) criado na primeira resposta | Responde questionários, acumula pontos, resgata cupons |

Princípio de privacidade (LGPD): o visitante nunca informa nome, e-mail ou telefone. Só o **hash** do token dele é guardado, e toda análise exibida ao lojista é agregada e protegida por um tamanho mínimo de grupo.

---

## 2. Arquitetura

```
┌──────────────────────────┐        HTTPS/JSON         ┌───────────────────────────────┐
│  Frontend (Next.js)      │ ────────────────────────► │  Backend (FastAPI, async)     │
│  App Router + Tailwind   │   /api/v1/...             │  Pydantic v2 + SQLAlchemy 2   │
│  Recharts, lucide-react  │ ◄──────────────────────── │  JWT (HS256) + Argon2         │
└──────────────────────────┘                           └──────────┬────────────┬───────┘
   localStorage:                                                  │            │ (opcional)
   • vitrineia.token (JWT lojista)                                ▼            ▼
   • vitrineia.visitante (token anônimo)                  Banco relacional   API Anthropic
                                                          (DATABASE_URL)    (só números agregados)
```

### Stack

**Backend**
- Python, FastAPI, SQLAlchemy 2.x assíncrono (`AsyncSession`), Pydantic v2 / `pydantic-settings`
- Autenticação: `PyJWT` (HS256) e `pwdlib` com Argon2 (`PasswordHash.recommended()`)
- Cliente HTTP assíncrono (`httpx`) para a IA generativa (opcional)

**Frontend**
- Next.js (App Router, componentes cliente), React, TypeScript
- Tailwind CSS, Recharts (gráficos), lucide-react (ícones)
- Sem biblioteca de estado global: `useFetch` (hook próprio) e `AuthProvider` (Context)

### Convenção de nomes
- O backend usa nomes de domínio em português (`estabelecimentos`, `vendas`, `cupons`...).
- Os schemas de resposta do painel usam inglês e **camelCase** via `CamelModel` (`alias_generator=to_camel`), espelhando os tipos TypeScript de `src/types`.
- Exceção: `POST /vendas` e `POST /auth/registro` usam **snake_case** / campos em português sem alias (ver `VendaCreate`, `EstabelecimentoCreate`).

---

## 3. Estrutura do projeto

### Backend (`app/`)
```
core/config.py          Settings (.env): banco, chave JWT, CORS, fuso, min_group_size, IA
core/security.py        Hash de senha, JWT, token anônimo do visitante (hash SHA-256)
db/base.py              Base declarativa + coluna criado_em
db/session.py           Engine/sessão assíncronas e dependência get_db
models/                 Estabelecimento, Venda, Campanha, Oportunidade, Questionario,
                        Pergunta, Opcao, Resposta, RespostaItem, Visitante, Cupom, Resgate
schemas/                Contratos Pydantic (camelCase para o frontend)
services/analytics.py   Cestas, segmentos, regras de associação, classificação
services/dashboard.py   Agregações do painel (KPIs, série mensal, categorias, atividade)
services/ia.py          Refinamento de texto com IA generativa (opcional)
services/sexta13.py     Regras do evento Sexta 13
services/tempo.py       "hoje" no fuso configurado + níveis de fidelidade
services/formatacao.py  Formatação de datas, moeda e tempo relativo
api/deps.py             Dependências: lojista autenticado, visitante opcional, sessão
api/v1/routes/*.py      Rotas (auth, vendas, cupons, rewards, dashboard, campaigns,
                        consumers, opportunities, eventos, questionarios)
```

### Frontend (`src/`)
```
app/                    Páginas (App Router)
  page.tsx              Visão geral (dashboard)
  login/, registro/     Autenticação do lojista
  consumidores/         Segmentos agregados
  oportunidades/        Insights + geração com IA
  campanhas/            Lista e criação de campanhas
  questionarios/        Lista, criação ("novo") e resultados ([id])
  cupons/               Criação de cupons + validação de resgate no balcão
  vendas/               Registro de vendas
  responder/[id]/       Página pública que o visitante responde (sem login)
  rewards/              Pontos e cupons do visitante (sem login)
components/             app-shell (layout + guarda de rota), auth-layout, form, ui
lib/api.ts              Cliente HTTP único, tokens, tratamento de erros
lib/auth.tsx            AuthProvider / useAuth
lib/hooks.ts            useFetch, errorMessage
types/index.ts          Tipos espelhando os schemas do backend
```

---

## 4. Modelo de dados

| Tabela | Campos principais | Observações |
|---|---|---|
| `estabelecimentos` | id, nome, segmento, **email (único)**, senha_hash | Lojista; `segmento` ajuda a sugerir lojas parceiras |
| `vendas` | estabelecimento_id, data, produto, categoria, quantidade, valor_total (Numeric 12,2) | Base do faturamento, ticket médio e vendas por categoria |
| `campanhas` | estabelecimento_id, nome, tipo, desconto, data_inicio, data_fim, alcance | Status é **calculado** pela data (Agendada/Ativa/Encerrada) |
| `oportunidades` | estabelecimento_id, titulo, descricao, lojas_sugeridas (JSON), potencial, confianca, categoria, prioridade | Regravada a cada análise |
| `questionarios` | estabelecimento_id, titulo, descricao, ativo, **pontos** | Pontos pagos ao visitante por resposta |
| `perguntas` | questionario_id, texto, tipo (`unica`/`multipla`), **dimensao** (`categoria`/`frequencia`/nula), ordem | Só `dimensao="categoria"` alimenta a análise |
| `opcoes` | pergunta_id, texto, ordem | |
| `visitantes` | **token_hash (único)**, pontos | Sem dados pessoais; guarda só o hash SHA-256 do token |
| `respostas` | questionario_id, visitante_id | **Única por (questionário, visitante)** |
| `resposta_itens` | resposta_id, pergunta_id, opcao_id | Uma linha por opção marcada |
| `cupons` | estabelecimento_id, campanha_id?, titulo, descricao, desconto, categoria, **codigo (único)**, **custo_pontos**, estoque (nulo = ilimitado), validade, ativo, exclusivo_sexta_13 | |
| `resgates` | cupom_id, visitante_id, **codigo (único)**, usado, usado_em | Código gerado: `{CODIGO_CUPOM}-{6 hex}` |

Todas as tabelas têm `criado_em` (`timestamptz`, padrão `now()`). Questionário → perguntas → opções usam `cascade="all, delete-orphan"`.

---

## 5. Identidade e autenticação

### 5.1 Lojista
1. `POST /auth/registro` cria o estabelecimento (senha com Argon2, e-mail único → `409` se repetido).
2. `POST /auth/login` (formulário `application/x-www-form-urlencoded`, campos `username` = e-mail e `password`) devolve `{access_token, token_type}`.
3. O JWT (`sub` = id, `exp` = 60 min por padrão) é enviado em `Authorization: Bearer`.
4. O frontend guarda o token em `localStorage` e valida a sessão com `GET /auth/me`. Qualquer `401` em rota de lojista limpa o token e dispara o redirecionamento para `/login?next=...`.

### 5.2 Visitante anônimo
1. Na **primeira resposta** a um questionário o backend gera um token (`secrets.token_urlsafe(32)`), grava só `sha256(token)` em `visitantes` e devolve o token **uma única vez** em `visitanteToken`.
2. O frontend guarda o token no `localStorage` do navegador e o envia no cabeçalho `X-Visitante-Token` nas chamadas seguintes (`/rewards`, `/coupons/redeem`, novas respostas).
3. Os **pontos pertencem a esse token**, ou seja, ao navegador. Não há login, e-mail ou recuperação de conta.

> **Consequência importante:** limpar os dados do navegador, usar aba anônima ou trocar de aparelho cria um visitante novo, com saldo zero. Os pontos e os códigos resgatados **não são portáveis** entre dispositivos. É uma troca consciente de conveniência por privacidade.

---

## 6. Fluxos de negócio

### 6.1 Questionário → pontos
1. Lojista cria o questionário (`POST /questionarios`) com 1–15 perguntas, 2–12 opções cada, `pontos` de 1 a 100.
2. Compartilha o link `/responder/{id}`.
3. Visitante abre o link, responde **todas** as perguntas (validação no servidor: pergunta válida, sem repetição, opções pertencentes à pergunta, uma opção apenas em `unica`).
4. `POST /public/questionarios/{id}/respostas` grava a resposta, cria o visitante se necessário e soma os pontos (`pontos × 2` em Sexta 13).
5. Uma segunda tentativa do mesmo visitante retorna `409` (reforçada por `UniqueConstraint` no banco, que também protege contra corrida).

### 6.2 Resgate de cupom
1. `GET /rewards` lista os cupons disponíveis e o saldo do visitante (sem token: saldo 0).
2. `POST /coupons/redeem` com o `code` do cupom:
   - exige token de visitante (senão `401` "Responda um questionário...");
   - **bloqueia as linhas do cupom e do saldo** (`SELECT ... FOR UPDATE`) para impedir resgate duplo ou estoque negativo;
   - valida existência, ativo, validade, regra da Sexta 13, estoque e saldo;
   - debita pontos, decrementa estoque e cria o `Resgate` com código único.
3. O visitante apresenta o código na loja; o lojista o confirma em `POST /resgates/validar`. Cada código só vale uma vez (`409` se repetido) e só o dono do cupom consegue validá-lo.

#### Regras de visibilidade na vitrine (`GET /rewards`)
Um cupom só aparece para o visitante quando **todas** as condições abaixo forem verdadeiras:

| Condição | Detalhe |
|---|---|
| `ativo = true` | |
| `validade >= hoje` | "hoje" no fuso `America/Sao_Paulo` (configurável) |
| Estoque disponível | `estoque` nulo (ilimitado) ou maior que zero |
| Não é secreto, **ou** hoje é Sexta 13 | Cupons com `exclusivo_sexta_13` ficam ocultos nos demais dias |

Se a vitrine estiver vazia, o motivo é quase sempre uma destas quatro condições (cupom expirado, pausado, esgotado ou marcado como oferta secreta de Sexta 13).

### 6.3 Níveis de fidelidade
`Explorador` (0 pts) → `Caçador de ofertas` (300) → `Mestre das vitrines` (1000). A barra de progresso mostra os pontos até o próximo nível.

### 6.4 Sexta 13
- `eh_sexta_13(d)`: dia 13 e sexta-feira.
- Nesse dia: pontos em dobro nos questionários, cupons secretos visíveis/resgatáveis e a campanha automática "Sexta 13 — Ofertas Secretas" aparece como **Ativa** (nos outros dias, como **Agendada** com a data da próxima).
- `GET /eventos/sexta-13` (público) alimenta o banner do frontend.

---

## 7. Análise de dados e IA

### 7.1 Cestas
Cada **submissão** de questionário vira uma "cesta": o conjunto de opções marcadas em perguntas com `dimensao = "categoria"` (texto normalizado com `casefold`).

### 7.2 Segmentos (`GET /consumers/segments`)
Para cada interesse: contagem, percentual sobre o total de seleções, **tendência** (variação % entre os últimos 30 dias e os 30 anteriores) e até 3 interesses mais frequentes em conjunto. Interesses com menos de `min_group_size` ocorrências são descartados.

### 7.3 Regras de associação (`POST /opportunities/gerar`)
Para cada par ordenado (A → B):

- **suporte** = ocorrências(A,B) / total de cestas
- **confiança** = ocorrências(A,B) / ocorrências(A)
- **lift** = confiança / (ocorrências(B) / total)

Mantém só pares com `ocorrências ≥ min_group_size` e `lift > 1`; para cada par não ordenado fica a direção de maior confiança; ordena por `lift × confiança` e usa as 5 melhores.

Classificação:

| Prioridade | Critério | Potencial |
|---|---|---|
| Alta | confiança ≥ 0,7 **e** lift ≥ 1,2 | Alto potencial |
| Média | confiança ≥ 0,5 | Potencial moderado |
| Baixa | demais | Potencial a validar |

"Lojas sugeridas" são estabelecimentos **parceiros** (outros lojistas) cujo `segmento` casa com algum dos dois interesses da regra (até 3).

### 7.4 IA generativa (opcional)
- Ativada apenas se `AI_API_KEY` estiver configurada; o modelo vem de `AI_MODEL`.
- Envia **somente números agregados** (interesses, confiança, lift, nº de respostas e segmento da loja), nunca respostas individuais.
- Prompt de sistema limita a resposta a 2 frases em pt-BR e proíbe inventar dados.
- Timeout configurável; se faltar chave, ocorrer erro de rede/HTTP ou resposta vazia, o sistema usa o **texto padrão por regras** (`descricao_padrao`). A funcionalidade nunca fica indisponível por causa da IA.

### 7.5 Painel (`GET /dashboard`)
- KPIs dos **últimos 30 dias** comparados aos 30 anteriores: faturamento, nº de vendas, ticket médio, consumidores ativos (visitantes distintos que responderam questionários da loja).
- Série mensal dos últimos 7 meses; distribuição por categoria (top 4 + "Outros"); atividade recente (vendas, respostas, resgates, campanhas) ordenada no tempo.
- Todos os dados de venda são filtrados pelo estabelecimento logado.

---

## 8. Privacidade e segurança

| Tema | Medida |
|---|---|
| **LGPD / anonimato** | Visitante sem dados pessoais; só `sha256(token)` no banco |
| **k-anonimato** | `min_group_size` (padrão 5): segmentos, regras e resultados de questionário com menos respostas não são exibidos (`suprimido=true`, valores `null`) |
| **Dados enviados à IA** | Apenas estatísticas agregadas |
| **Senhas** | Argon2; mínimo de 8 caracteres |
| **Isolamento entre lojistas** | Toda consulta/ação filtra por `estabelecimento_id`; recurso de outro lojista retorna `404` |
| **Concorrência** | `FOR UPDATE` no resgate e na validação; `UNIQUE (questionario_id, visitante_id)` |
| **Validação de entrada** | Pydantic (limites de tamanho, `gt/ge/le`, `data_fim ≥ data_inicio`) e validação extra das respostas |
| **Redirecionamento** | O parâmetro `next` do login só aceita caminhos internos (`/...`, nunca `//`) |
| **Erros** | O cliente traduz `detail` do FastAPI (string ou lista 422) em mensagem legível e trata falha de rede |

---

## 9. Referência da API

Prefixo: `/api/v1`. "🔒 Lojista" = `Authorization: Bearer`; "👤 Visitante" = `X-Visitante-Token` (opcional ou obrigatório conforme indicado).

### Autenticação
| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| POST | `/auth/registro` | Público | Cria estabelecimento (`nome, segmento, email, senha`) |
| POST | `/auth/login` | Público | Form `username`/`password` → JWT |
| GET | `/auth/me` | 🔒 | Dados do lojista logado |

### Painel e análise
| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| GET | `/dashboard` | 🔒 | KPIs, séries e atividade |
| GET | `/consumers/segments` | 🔒 | Segmentos agregados e anônimos |
| GET | `/opportunities` | 🔒 | Oportunidades salvas (por confiança) |
| POST | `/opportunities/gerar` | 🔒 | Recalcula regras, chama IA (se houver) e regrava |

### Operação do lojista
| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| POST | `/vendas` | 🔒 | Registra venda (`data, produto, categoria?, quantidade, valor_total`) |
| GET / POST | `/campaigns` | 🔒 | Lista (inclui a campanha automática Sexta 13) / cria |
| GET / POST | `/cupons` | 🔒 | Lista / cria cupom (código gerado no servidor) |
| POST | `/resgates/validar` | 🔒 | Confirma o código apresentado no balcão |

### Questionários
| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| POST | `/questionarios` | 🔒 | Cria questionário com perguntas e opções |
| GET | `/questionarios` | 🔒 | Lista com total de respostas |
| PATCH | `/questionarios/{id}` | 🔒 | Ativa/pausa (`{ativo}`) |
| GET | `/questionarios/{id}/resultados` | 🔒 | Distribuição por opção (suprimida com poucas respostas) |
| GET | `/public/questionarios/{id}` | Público | Questionário ativo para o visitante responder |
| POST | `/public/questionarios/{id}/respostas` | 👤 opcional | Registra resposta; devolve pontos e, na 1ª vez, `visitanteToken` |

### Visitante e eventos
| Método | Rota | Acesso | Descrição |
|---|---|---|---|
| GET | `/rewards` | 👤 opcional | Saldo, nível e cupons disponíveis (sem token: 0 pontos) |
| POST | `/coupons/redeem` | 👤 **obrigatório** | Troca pontos por cupom e gera o código de resgate |
| GET | `/eventos/sexta-13` | Público | Estado do evento e próxima data |

Códigos comuns: `401` credencial/visitante ausente, `403` oferta secreta fora da Sexta 13, `404` não encontrado/indisponível, `409` conflito (e-mail repetido, já respondeu, esgotado, já utilizado), `400` pontos insuficientes, `422` validação.

---

## 10. Configuração e execução

### Backend (`.env`)
| Variável | Obrigatória | Padrão | Uso |
|---|---|---|---|
| `DATABASE_URL` | Sim | — | URL do banco com driver assíncrono |
| `SECRET_KEY` | Sim | — | Assinatura do JWT (use valor longo e aleatório) |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Não | 60 | Duração do JWT |
| `CORS_ORIGINS` | Não | `["http://localhost:3000"]` | Origens permitidas |
| `TIMEZONE` | Não | `America/Sao_Paulo` | Define o "hoje" (validade, Sexta 13) |
| `MIN_GROUP_SIZE` | Não | 5 | Limite de k-anonimato |
| `AI_API_KEY` | Não | vazio | Liga a IA generativa |
| `AI_MODEL` | Não | `claude-sonnet-5-5` | Modelo usado nas recomendações |
| `AI_TIMEOUT_SECONDS` | Não | 25 | Timeout da chamada de IA |

> O CORS precisa permitir o cabeçalho **`X-Visitante-Token`** (por exemplo `allow_headers=["*"]`), senão o navegador bloqueia as chamadas do visitante.

### Frontend (`.env.local`)
```
NEXT_PUBLIC_API_URL=http://localhost:8000/api/v1
```

### Subir o ambiente
```bash
# backend (ajuste o módulo conforme o main.py do projeto)
uvicorn app.main:app --reload --port 8000

# frontend
npm install
npm run dev        # http://localhost:3000
```

---

## 11. Roteiro de teste ponta a ponta

1. **Cadastrar loja** em `/registro` (entra logado).
2. **Criar questionário** em `/questionarios/novo` com "Usar modelo de interesses" e publicar.
3. **Criar cupom** em `/cupons` com validade futura e **sem** marcar "Sexta 13".
4. **Responder como visitante**: abrir `/responder/1` em outra janela/perfil e enviar → recebe pontos.
5. Em `/rewards` (mesmo navegador do passo 4) **resgatar** o cupom → recebe o código.
6. Em `/cupons`, colar o código em **Validar resgate** (como lojista).
7. Repetir o passo 4 com ao menos 5 visitantes distintos (navegadores/perfis diferentes) para liberar **Consumidores**, **resultados do questionário** e **Oportunidades**.

---

## 12. Limitações conhecidas e próximos passos

| # | Limitação | Impacto | Sugestão |
|---|---|---|---|
| 1 | Pontos presos ao navegador (sem login do visitante) | Perda de saldo ao limpar dados/trocar de aparelho; reset fácil burla o "1 resposta por pessoa" | Login opcional por telefone/e-mail com consentimento, ou QR code de carteira |
| 2 | `GET /rewards` não devolve `custo_pontos` | O visitante não vê quanto cada cupom custa | Adicionar `cost_points` ao schema `Reward` |
| 3 | Não há endpoint público para listar questionários | O visitante só chega a uma pesquisa por link/QR | Listar pesquisas ativas por loja/shopping |
| 4 | Não há listagem de resgates do visitante | Código resgatado só é visto na hora | Guardar histórico no aparelho ou endpoint de carteira |
| 5 | Painel do lojista não explica por que um cupom não aparece na vitrine | Confusão com cupons expirados, pausados, esgotados ou secretos | Mostrar o status de visibilidade em cada cupom e permitir pausar/reativar |
| 6 | JWT em `localStorage` | Exposto a XSS | Cookie `HttpOnly` + CSRF, ou refresh token |
| 7 | Sem *rate limiting* nem proteção anti-bot nos endpoints públicos | Possível abuso de pontos | Limite por IP/token, CAPTCHA |
| 8 | Lojas parceiras sugeridas por casamento de texto entre `segmento` e interesse | Sugestões imprecisas | Taxonomia de categorias compartilhada |
| 9 | Sem testes automatizados e migrações versionadas documentadas | Risco de regressão | Pytest (regras de análise, Sexta 13, resgate) e Alembic |
| 10 | Campos como `reach` de campanha são informados manualmente | Alcance não é medido | Medir por visualizações/respostas |