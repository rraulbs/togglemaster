# Análise da aplicação monolítica ToggleMaster

Esta análise se baseia no código-fonte da aplicação (`app.py`, `Dockerfile`, `docker-compose.yaml`, `entrypoint.sh` e `requirements.txt`).

---

## 1. Como a aplicação funciona

### 1.1 Tecnologias usadas

| Camada | Tecnologia | Papel |
|---|---|---|
| Linguagem | Python 3.9 (imagem `python:3.9-slim`) | Runtime |
| Framework web | Flask 2.2.2 | Rotas HTTP e JSON |
| Servidor de aplicação | Gunicorn 20.1.0 | Servidor WSGI de produção (porta 5000) |
| Driver de banco | psycopg2-binary 2.9.5 | Conexão com PostgreSQL |
| Banco de dados | PostgreSQL 13 (container no compose; RDS na AWS) | Persistência |
| Empacotamento | Docker + Docker Compose | Execução local |

### 1.2 Estrutura do código

- `app.py` concentra toda a aplicação: configuração, conexão com o banco, criação da tabela, rotas e validações.
- `entrypoint.sh` valida as variáveis de ambiente, espera o banco ficar pronto com `pg_isready`, executa `flask init-db` e inicia o Gunicorn.
- `Dockerfile` empacota a aplicação e instala o `postgresql-client`, necessário para o `pg_isready`.
- `docker-compose.yaml` sobe a aplicação e um PostgreSQL local com healthcheck.

### 1.3 Modelo de dados

Existe uma única tabela, `flags`:

| Coluna | Tipo | Observação |
|---|---|---|
| `id` | SERIAL, PK | |
| `name` | VARCHAR(100) | UNIQUE, NOT NULL |
| `is_enabled` | BOOLEAN | NOT NULL, default `false` |
| `created_at` | TIMESTAMPTZ | default `CURRENT_TIMESTAMP` |

### 1.4 Endpoints e fluxo básico

| Método | Rota | Comportamento | Respostas |
|---|---|---|---|
| GET | `/health` | Retorna `{"status":"ok"}` | 200 |
| POST | `/flags` | Cria flag (`name` obrigatório, `is_enabled` opcional) | 201, 400, 409 (duplicada), 500 |
| GET | `/flags` | Lista todas, ordenadas por nome | 200, 500 |
| GET | `/flags/<name>` | Consulta uma flag | 200, 404, 500 |
| PUT | `/flags/<name>` | Atualiza `is_enabled` (deve ser booleano) | 200, 400, 404, 500 |

**Fluxo de uma requisição (exemplo: `POST /flags`):**

1. O cliente envia `POST /flags` com `{"name": "new-feature", "is_enabled": true}`.
2. O Gunicorn recebe a requisição e a repassa ao Flask.
3. A rota `create_flag()` valida o JSON (`name` obrigatório).
4. A função `get_db_connection()` abre uma **nova conexão** com o PostgreSQL, usando as variáveis de ambiente.
5. A rota executa o `INSERT` com query parametrizada (`%s`), o que protege contra SQL injection.
6. O `commit` confirma a gravação. Se o nome já existir, ocorre `IntegrityError` e a resposta é 409.
7. O bloco `finally` fecha o cursor e a conexão.
8. A resposta JSON é devolvida ao cliente (201).

**Configuração:** `DB_HOST`, `DB_NAME`, `DB_USER` e `DB_PASSWORD` são lidas do ambiente.

### 1.5 Observações sobre o código

1. **Não existe endpoint DELETE.** O enunciado fala em CRUD completo (criar, ler, atualizar e deletar), mas a API implementa apenas C, R e U. Essa é uma limitação conhecida do MVP.
2. **O `/health` não verifica o banco.** Ele responde "ok" mesmo se o PostgreSQL estiver fora do ar, então é um *liveness check*, não um *readiness check*.
3. **Não há pool de conexões.** Cada requisição abre e fecha uma conexão. Funciona no MVP, mas é custoso sob carga.
4. **Os erros 500 devolvem `str(e)` ao cliente** (campo `details`), o que pode vazar detalhes internos.
5. **Não há autenticação.** Qualquer cliente que alcance a API pode criar ou alterar flags.
6. **As mensagens do README estão em inglês** (`created successfully`), mas o código responde em português (`criada com sucesso`). É apenas uma divergência de documentação.

---

## 2. Por que a aplicação é considerada um monólito

Um monólito é uma aplicação em que **toda a lógica vive em uma única base de código e é implantada como uma única unidade**.

### 2.1 Evidências no projeto

1. **Código único e sem separação de camadas.** No `app.py`, a mesma função faz roteamento HTTP, validação, regra de negócio e acesso a dados (o SQL é escrito dentro da própria rota). Não há módulos, serviços ou repositórios separados.
2. **Unidade única de deploy.** Existe um só artefato (uma imagem Docker ou um processo Gunicorn na EC2). Para alterar qualquer coisa, como uma validação ou uma query, é preciso reimplantar a aplicação inteira.
3. **Escalabilidade apenas em bloco.** Para escalar, é necessário replicar a aplicação toda. Não é possível escalar apenas a leitura ou apenas a escrita.
4. **Stack única e banco compartilhado.** Todo o domínio usa a mesma tecnologia (Python/Flask) e o mesmo schema PostgreSQL.
5. **Ciclo de vida único.** Uma falha que derrube o processo derruba todas as funcionalidades ao mesmo tempo.

### 2.2 O papel do banco de dados

O banco de dados é um componente separado da aplicação: no Docker Compose é o container `db`, e na AWS será o RDS. Isso é a prática normal e **não descaracteriza o monólito**, porque o termo se refere à aplicação, não à infraestrutura ao redor. O ponto central é que toda a lógica da aplicação está em um único serviço:

> A aplicação é um monólito porque toda a lógica (API, validação, regras de negócio e acesso a dados) está em uma única base de código e é implantada como uma única unidade. O banco de dados é um componente externo, mas a aplicação como um todo não pode ser evoluída, escalada ou implantada em partes.

Em uma arquitetura de microsserviços, o ToggleMaster poderia ser dividido, por exemplo, em um serviço de gestão de flags, um de autenticação e um de avaliação de flags, cada um com seu próprio deploy e, muitas vezes, seu próprio banco.

---

## 3. Vantagens e desvantagens para um MVP

### 3.1 Vantagens

| Vantagem | Como aparece no ToggleMaster |
|---|---|
| **Simplicidade** | Um único arquivo e um fluxo fácil de entender e depurar |
| **Velocidade de desenvolvimento** | Uma API CRUD sobe rápido, sem contratos entre serviços |
| **Deploy simples** | Um artefato e um processo; na EC2, basta clonar, instalar e rodar |
| **Menos coordenação** | Um time pequeno trabalha em uma só base de código |
| **Menor custo inicial** | Uma EC2 pequena e um RDS pequeno bastam |
| **Consistência de dados** | Um único banco, sem transações distribuídas |
| **Menos overhead operacional** | Não há service discovery, mensageria nem tracing distribuído |
| **Validação rápida da ideia** | É o objetivo do MVP: colocar no ar e receber feedback |

### 3.2 Desvantagens

| Desvantagem | Impacto no ToggleMaster |
|---|---|
| **Escalabilidade apenas em bloco** | Se a leitura de flags for muito mais frequente que a escrita, não há como escalar só a leitura |
| **Acoplamento** | Regra de negócio, HTTP e SQL misturados dificultam testes e mudanças |
| **Deploy de risco** | Uma pequena alteração exige reimplantar tudo |
| **Ponto único de falha** | Uma EC2 e um processo: se cair, tudo cai |
| **Dificuldade de evoluir para microsserviços** | Antes de dividir, é preciso separar camadas e responsabilidades |
| **Stack única** | Todo o sistema fica preso a Python/Flask |
| **Crescimento da complexidade** | Com mais funcionalidades, o `app.py` tende a virar um arquivo difícil de manter |
| **Escalabilidade de equipe limitada** | Vários times mexendo na mesma base geram conflitos e dependências |

### 3.3 Conclusão

Para um MVP, o monólito é a escolha adequada: ele reduz o tempo até o primeiro deploy, a complexidade e o custo, que é o que importa para validar a ideia. As desvantagens (escalabilidade independente, acoplamento, ponto único de falha) só se tornam relevantes quando o produto cresce em usuários e funcionalidades. Por isso a evolução natural é começar monolítico e dividir em serviços quando houver necessidade comprovada, o que está alinhado à proposta do projeto de transformá-lo gradualmente em um sistema distribuído ao longo das próximas fases.