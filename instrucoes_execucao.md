
# 🚀 VitrineIA — Plataforma de Inteligência Comercial e Engajamento

<p align="left">
  <img src="https://img.shields.io/badge/Status-Em%20Desenvolvimento-blue" alt="Status">
  <img src="https://img.shields.io/badge/Docker-Enabled-2496ED?logo=docker&logoColor=white" alt="Docker">
  <img src="https://img.shields.io/badge/FastAPI-0.9+-005571?logo=fastapi&logoColor=white" alt="FastAPI">
  <img src="https://img.shields.io/badge/Next.js-15-black?logo=next.js&logoColor=white" alt="Next.js">
</p>

Plataforma centralizada de inteligência comercial e engajamento de lojistas voltada para centros de compras, integrada com painéis gerenciais, questionários dinâmicos e automação baseada em Inteligência Artificial.

---

## 🛠️ Pré-requisitos do Sistema

Antes de iniciar a configuração do ambiente, certifique-se de possuir as seguintes ferramentas instaladas em sua máquina:

* **Git** (`git --version`)
* **Docker** (`docker --version`)
* **Docker Compose** (`docker compose version`)

> **Nota para Linux (Ubuntu/Debian):** Certifique-se de que o daemon do Docker está ativo (`sudo systemctl start docker`) e que seu usuário possui permissões para executá-lo sem o prefixo `sudo`.

---

## ⚙️ Configuração e Instalação do Ambiente

Siga o passo a passo abaixo para clonar, configurar e executar a aplicação completa via Docker Compose.

### 1. Clonar o Repositório
Abra o seu terminal, clone o repositório oficial e navegue até a pasta raiz:

```bash
git clone https://github.com/Charizard-Rackarun/infra.git
cd infra

```

### 2. Configurar as Variáveis de Ambiente (`.env`)

O projeto disponibiliza arquivos de modelo para as variáveis de ambiente do microsserviço de backend e frontend. Duplique-os e renomeie-os conforme o seu ambiente operacional:

* **Linux / macOS / Git Bash:**
```bash
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env

```


* **Windows (PowerShell):**
```powershell
copy backend\.env.example backend\.env
copy frontend\.env.example frontend\.env

```



> ⚠️ **Importante:** Revise os arquivos `backend/.env` e `frontend/.env` recém-criados e ajuste chaves secretas, portas ou strings de conexão conforme necessário antes de prosseguir.

### 3. Build e Execução dos Containers

Na pasta raiz do projeto, execute o comando do Docker Compose para compilar as imagens e inicializar os serviços em segundo plano (*detached mode*):

```bash
docker compose up -d --build

```

* `--build`: Assegura a recompilação limpa das imagens do frontend e backend utilizando as dependências mais recentes.
* `-d`: Mantém os containers executando em segundo plano, liberando o terminal.

### 4. Execução das Migrações do Banco de Dados (Alembic)

Com os containers em execução, aplique as migrações estruturais do banco de dados relacional via ORM (Alembic) para garantir que todas as tabelas e colunas estejam criadas:

```bash
docker compose exec backend alembic revision --autogenerate -m "Atualiza o banco"
docker compose exec backend alembic upgrade head

```

---

## 🌐 Endpoints e Acessos da Aplicação

Após a inicialização bem-sucedida, os serviços estarão acessíveis nas seguintes portas locais:

| Serviço | URL de Acesso | Descrição |
| --- | --- | --- |
| **Frontend (Aplicação Web)** | `http://localhost:3000` | Interface do usuário e painel lojista |
| **Backend (API / Documentação)** | `http://localhost:8000/docs` | Documentação interativa (Swagger UI) |

---

## 📦 Comandos Úteis para Gestão (Docker)

Gerencie o ciclo de vida da infraestrutura utilizando os comandos abaixo na raiz do projeto:

* **Verificar o status dos containers:**
```bash
docker compose ps

```


* **Monitorar logs em tempo real (todos os serviços):**
```bash
docker compose logs -f

```


* **Monitorar logs de um serviço específico:**
```bash
docker compose logs -f backend
# ou
docker compose logs -f frontend

```


* **Parar os containers de forma segura:**
```bash
docker compose stop

```


* **Parar e remover os containers e redes criadas:**
```bash
docker compose down

```


*(Dica: Adicione a flag `-v` — `docker compose down -v` — caso deseje limpar completamente os volumes e resetar o banco de dados).*

---

## 📂 Arquitetura do Repositório

```text
infra/
├── backend/                 # Microsserviço API (Python / FastAPI / Alembic)
│   ├── alembic/             # Histórico de migrações do banco de dados
│   ├── app/                 # Regras de negócios, rotas e modelos ORM
│   ├── .env.example         # Template de variáveis de ambiente do backend
│   ├── Dockerfile           # Configuração de containerização do backend
│   └── requirements.txt     # Dependências do ecosistema Python
├── frontend/                # Aplicação Client-Side (Next.js 15 / TypeScript)
│   ├── src/                 # Páginas, componentes estruturais e hooks
│   ├── .env.example         # Template de variáveis de ambiente do frontend
│   ├── Dockerfile           # Configuração de containerização do frontend
│   └── package.json         # Dependências do ecossistema Node.js
├── .gitignore               # Arquivos e diretórios ignorados pelo controle de versão
├── docker-compose.yml       # Orquestração de containers (DB PostgreSQL, API, Web)
└── README.md                # Documentação técnica do projeto

```