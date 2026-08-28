# Desafio-Pipeline-CI-CD-Completo-com-GitHub-Actions

API simples em FastAPI com uma pipeline de CI/CD completa feita no GitHub Actions, seguindo GitFlow. Esse projeto foi feito como desafio do meu estágio em CloudOps, com o objetivo de simular o fluxo de trabalho de uma equipe de DevOps: testes automatizados, análise de segurança estática, build e publicação de imagem Docker.

## O que a API faz

São só 2 endpoints:

- `GET /` — retorna uma mensagem de saudação
- `GET /health` — retorna o status da aplicação

## Rodando localmente

Clone o repositório, entre na pasta e crie um ambiente virtual (recomendo, pra não bagunçar seu Python global):

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Depois é só subir a API:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

E acessar `http://localhost:8000` e `http://localhost:8000/health`.

Pra rodar os testes:

```bash
python -m pytest tests/ -v
```

Só rodar `pytest tests/ -v` direto dá erro (`ModuleNotFoundError: No module named 'app'`), porque sem um `__init__.py` em `tests/` o pytest não acha a raiz do projeto no path e não localiza o pacote `app`. Rodando como `python -m pytest`, o Python adiciona a pasta atual ao path e o `from app.main import app` funciona.

## Rodando com Docker

```bash
docker build -t cloudops-api .
docker run -p 8000:8000 cloudops-api
```

## Como funciona a pipeline

O projeto segue GitFlow: `main` (produção, protegida), `develop` (integração) e `feature/*` (desenvolvimento). Toda mudança nasce numa `feature/*`, vai pra `develop` via PR, e só depois de validada segue pra `main`.

Tem dois workflows separados no `.github/workflows/`:

**`ci.yml`** — dispara em todo Pull Request pra `develop` e `main`. Roda dois jobs em paralelo:
- `test`: instala as dependências e roda os testes com pytest. Se algum teste falhar, o job falha e o merge fica bloqueado (branch protection configurada em `main` e `develop` exige esse check passando).
- `sast`: roda o Semgrep com o ruleset `p/python`, pra pegar possíveis vulnerabilidades no código. Esse não bloqueia o merge, só reporta nos logs.

**`cd.yml`** — dispara só quando tem push direto na `main` (ou seja, só depois que um PR é mergeado nela). Faz login no Docker Hub usando secrets, builda a imagem a partir do Dockerfile e publica com duas tags: `latest` e o SHA do commit.

```
PR feature/* -> develop   ==>  roda ci.yml (test + sast)
PR develop -> main        ==>  roda ci.yml (test + sast)
merge na main             ==>  roda cd.yml (build + push Docker Hub)
```

As credenciais do Docker Hub ficam guardadas como GitHub Secrets (`DOCKERHUB_USERNAME` e `DOCKERHUB_TOKEN`), nunca aparecem no código nem nos logs (o GitHub mascara automaticamente).

## Estrutura

```
.github/workflows/
  ci.yml
  cd.yml
app/
  __init__.py
  main.py
tests/
  test_main.py
Dockerfile
requirements.txt
```