# 12-Factor App aplicado ao ToggleMaster

Esta análise avalia a aplicação monolítica ToggleMaster (`app.py`, `Dockerfile`, `docker-compose.yaml`, `entrypoint.sh` e `requirements.txt`) frente aos 12 fatores da metodologia [12-Factor App](https://12factor.net/pt_br/). Para cada fator, apresentamos a evidência no código, o status e a melhoria sugerida para um ambiente de produção.

Na Fase 1, a aplicação é implantada na AWS **como está**: o código (`app.py`) não foi alterado, e a única mudança no diretório `toggle-master-monolith/` foi a imagem do banco local no `docker-compose.yaml` (`postgres:13` para `postgres:16`), para manter a paridade com o RDS. As melhorias descritas aqui são recomendações para um ambiente de produção mais robusto e não fazem parte do escopo desta fase.

A análise foi atualizada após a implantação, de modo que as evidências refletem o que foi efetivamente provisionado (detalhes em `03-arquitetura-aws.md`).

**Legenda:** ✅ atende | ⚠️ atende parcialmente | ❌ não atende

---

## 1. Visão geral

| # | Fator | Status | Resumo |
|---|---|---|---|
| I | Codebase | ✅ | Uma base de código versionada em Git, com deploys local e AWS |
| II | Dependências | ✅ | Declaradas e fixadas em `requirements.txt`; há uma dependência de sistema implícita |
| III | Configuração | ⚠️ | Variáveis de ambiente e, na AWS, segredos no Secrets Manager; restam inconsistências (`DB_PORT`, SSL, validação) |
| IV | Serviços de apoio | ✅ | O banco é um recurso anexado, trocável apenas por variáveis de ambiente |
| V | Build, release, run | ❌ | Deploy manual na EC2; sem artefato versionado nem pipeline |
| VI | Processos | ✅ | Aplicação stateless; todo o estado fica no PostgreSQL |
| VII | Vínculo de porta | ✅ | Autocontida: o Gunicorn expõe a porta 5000 |
| VIII | Concorrência | ⚠️ | Escala por processos, mas sem pool de conexões e com worker único por padrão |
| IX | Descartabilidade | ⚠️ | Inicia rápido e usa `exec`, mas sem gerenciador de processos na EC2 |
| X | Paridade dev/prod | ⚠️ | Mesmo motor, versão do banco e Python, mas local roda em Docker e a EC2 em venv |
| XI | Logs | ⚠️ | Sai em stdout/stderr, mas exceções não são registradas |
| XII | Processos administrativos | ✅ | `flask init-db` roda como comando avulso, com o mesmo código |

**Placar:** 6 fatores atendidos, 5 parcialmente e 1 não atendido.

---

## 2. Análise detalhada por fator

### I. Codebase — ✅ Atende

**Princípio:** uma base de código rastreada em controle de versão, com vários deploys.

**Evidência:** o projeto é um único repositório Git. O mesmo código roda localmente (Docker Compose) e rodará na EC2.

**Observação:** por ser um monólito, há uma única base para uma única aplicação, que é exatamente o que o fator pede. A divisão em vários serviços exigiria um repositório (ou módulo) por serviço.

---

### II. Dependências — ✅ Atende (com ressalvas)

**Princípio:** declarar e isolar explicitamente todas as dependências.

**Evidência:**
- `requirements.txt` com versões fixas: Flask 2.2.2, Werkzeug 2.3.8, psycopg2-binary 2.9.5 e gunicorn 20.1.0.
- Isolamento por container (Dockerfile) ou por ambiente virtual (`venv`) na EC2. Na implantação, o `pip install -r requirements.txt` no `venv` com Python 3.9.25 (Amazon Linux 2023) concluiu sem erros.

**Ressalvas:**
1. O `entrypoint.sh` depende do `pg_isready`, parte do pacote `postgresql-client`. No Docker ele é instalado pelo Dockerfile. Na EC2, só é necessário se o `entrypoint.sh` for usado; se o Gunicorn for iniciado diretamente (como foi feito), a dependência não se aplica. Em ambos os casos, **ela não está declarada fora do Dockerfile**.
2. `app.py` importa `click` diretamente, mas ele não está no `requirements.txt` (chega só como dependência transitiva do Flask). O import também não é usado.
3. Não há *lockfile* com dependências transitivas, e as versões fixadas são antigas (Flask 2.2.2, Python 3.9).

**Melhorias:**
- Documentar a instalação do `postgresql-client` caso o `entrypoint.sh` seja usado na EC2.
- Remover o `import click` não utilizado.
- Gerar um arquivo de versões completo (`pip freeze` ou `pip-tools`) e planejar a atualização de Python e Flask.

---

### III. Configuração — ⚠️ Atende parcialmente

**Princípio:** guardar a configuração no ambiente, separada do código.

**Evidência (pontos positivos):**
- `app.py` lê `DB_HOST`, `DB_NAME`, `DB_USER` e `DB_PASSWORD` com `os.getenv`. Nenhuma credencial está no código.
- A mesma imagem funciona em qualquer ambiente, mudando apenas as variáveis.
- Na AWS, `DB_USER` e `DB_PASSWORD` vêm do segredo gerenciado pelo RDS no **AWS Secrets Manager**, lido pela EC2 por meio de uma IAM Role (`LabInstanceProfile`) e exportado como variáveis de ambiente antes de iniciar o Gunicorn. A senha não é digitada, não fica no histórico do shell nem em arquivo, e o `app.py` não precisou ser alterado.

**Problemas encontrados:**
1. **Inconsistência com `DB_PORT`:** o `entrypoint.sh` exige essa variável e a usa no `pg_isready`, mas o `app.py` não a lê. O `psycopg2` acaba usando a porta padrão 5432. Funciona com RDS PostgreSQL, mas a configuração não é fiel ao que se declara.
2. **Validação incompleta:** o `entrypoint.sh` valida `DB_HOST`, `DB_PORT` e `DB_NAME`, mas não `DB_USER` e `DB_PASSWORD`. Além disso, com `set -u` ativo, uma variável ausente derruba o script com erro genérico antes de a mensagem amigável ser exibida.
3. **Gestão manual das variáveis:** o `docker-compose.yaml` versiona `user`/`password`, o que é aceitável em desenvolvimento local, mas não pode ser copiado para produção. Na EC2, a leitura do segredo é feita manualmente a cada sessão: as variáveis só existem no terminal em que foram exportadas (é preciso repetir o procedimento e reiniciar o Gunicorn ao reconectar), e a aplicação lê a senha uma única vez, ao iniciar. Como o segredo do RDS é rotacionado automaticamente (por padrão, a cada 7 dias), uma rotação exige reiniciar a aplicação.
4. **SSL sem validação de certificado:** a conexão não define `sslmode`, e o padrão do `psycopg2` (`prefer`) negocia SSL quando o servidor oferece. O RDS PostgreSQL 15 ou superior exige SSL por padrão, então o tráfego é criptografado, mas o certificado do servidor não é verificado.
5. **Porta da aplicação fixa:** `5000` está escrita no Dockerfile, no entrypoint e no `app.py`.

**Melhorias:**
- Ler `DB_PORT` no `app.py` e passá-la ao `psycopg2.connect`.
- Validar todas as variáveis obrigatórias com mensagens claras:
  ```bash
  : "${DB_HOST:?DB_HOST não definida}"
  : "${DB_PORT:?DB_PORT não definida}"
  : "${DB_NAME:?DB_NAME não definida}"
  : "${DB_USER:?DB_USER não definida}"
  : "${DB_PASSWORD:?DB_PASSWORD não definida}"
  ```
- Automatizar a leitura do segredo (por exemplo, no serviço `systemd` ou pela própria aplicação) e tratar a rotação sem reinício manual. O **Secrets Manager** já é usado na Fase 1; a evolução é deixar de depender de um passo manual.
- Usar `sslmode=verify-full` na conexão com o RDS, com o certificado raiz da AWS.
- Tornar a porta configurável (`PORT`).

---

### IV. Serviços de apoio — ✅ Atende

**Princípio:** tratar serviços de apoio (bancos, filas, caches) como recursos anexados.

**Evidência:** o banco é acessado exclusivamente por variáveis de ambiente. No Docker Compose, `DB_HOST=db` aponta para o container PostgreSQL. Na AWS, bastou ajustar as variáveis de conexão (`DB_HOST` com o endpoint do RDS, além de `DB_NAME`, `DB_USER` e `DB_PASSWORD`, conforme o banco criado), **sem alterar uma linha de código**, o que foi confirmado na implantação. É o melhor exemplo de aderência do projeto.

**Melhoria:** nenhuma obrigatória. Como boa prática, a mesma versão principal do PostgreSQL foi mantida no RDS e no compose (16). A versão 13, usada originalmente no compose, saiu do suporte padrão do RDS em 28/02/2026.

---

### V. Build, release, run — ❌ Não atende

**Princípio:** separar rigorosamente os estágios de construção, liberação e execução.

**Evidência:**
- **Build:** existe um Dockerfile, mas na AWS o código foi copiado para a EC2 (`scp`) e as dependências instaladas com `pip install` diretamente na instância, misturando build e run em uma operação manual.
- **Release:** não há versionamento de artefato (tags, imagem em registry). Não é possível apontar "a versão X está no ar" nem fazer rollback preciso.
- **Run:** a inicialização do banco (`flask init-db`) acontece na partida do container, o que mistura uma etapa de release (migração de schema) com a execução. Na EC2, ela foi executada manualmente, uma vez.
- Não há pipeline de CI/CD.

**Melhorias:**
- Versionar com tags Git e gerar imagens Docker com tag de versão, publicadas em um registry (Amazon ECR).
- Automatizar build e deploy com pipeline de CI/CD.
- Separar a migração de schema da partida da aplicação (etapa de release).
- Para a Fase 1, documentar o procedimento manual (feito em `03-arquitetura-aws.md`) e registrar o commit implantado.

---

### VI. Processos — ✅ Atende

**Princípio:** executar a aplicação como processos stateless, sem compartilhar estado.

**Evidência:** a aplicação não guarda sessão, cache em memória nem arquivos locais. Cada requisição abre sua própria conexão e todo estado persistente fica no PostgreSQL. Isso permite reiniciar ou replicar a aplicação sem perda de dados.

**Observação:** a ausência de autenticação também significa que não há sessão de usuário. Quando ela for adicionada, deve usar tokens ou um armazenamento externo, nunca memória local.

---

### VII. Vínculo de porta — ✅ Atende

**Princípio:** exportar serviços via vínculo de porta, sem depender de um servidor web externo.

**Evidência:** a aplicação é autocontida. O Gunicorn faz o *bind* em `0.0.0.0:5000` e atende HTTP diretamente, sem Apache ou Nginx obrigatórios. Na AWS, a API respondeu em `http://<ip-publico>:5000` com apenas a porta 5000 liberada no Security Group.

**Melhoria:** ler a porta de uma variável (`--bind 0.0.0.0:${PORT:-5000}`). Em produção, é comum colocar um proxy reverso ou Load Balancer na frente para TLS e portas 80/443.

---

### VIII. Concorrência — ⚠️ Atende parcialmente

**Princípio:** escalar horizontalmente por meio do modelo de processos.

**Evidência (positivo):** por ser stateless, a aplicação pode ser replicada em vários processos ou instâncias.

**Problemas:**
1. O Gunicorn é iniciado sem `--workers`, então roda com **um único worker** por padrão.
2. Não há *pool* de conexões: cada requisição abre e fecha uma conexão. Ao escalar workers ou instâncias, o número de conexões ao RDS cresce rápido e pode esgotar o limite do banco.

**Melhorias:**
- Definir `--workers` conforme o número de vCPUs (por exemplo, `2 x CPUs + 1`).
- Adotar pool de conexões (`psycopg2.pool`) ou um intermediário como o RDS Proxy.
- Planejar escala horizontal futura com Load Balancer e Auto Scaling.

---

### IX. Descartabilidade — ⚠️ Atende parcialmente

**Princípio:** processos rápidos para iniciar e que encerram de forma graciosa.

**Evidência (positivo):**
- A partida é rápida e o `entrypoint.sh` usa `exec gunicorn`, de modo que o Gunicorn recebe os sinais do sistema (SIGTERM) e encerra com graça.
- O Compose usa `depends_on` com *healthcheck* para esperar o banco.

**Problemas:**
1. O loop de espera do banco (`until pg_isready`) **não tem limite de tentativas**: se o banco nunca subir, o container fica preso para sempre.
2. Na EC2, o Gunicorn roda em primeiro plano; se a sessão SSH fecha, a aplicação cai, e não há reinício automático em caso de falha. Na prática, isso exige reiniciar a aplicação (e reexportar as variáveis) a cada nova sessão.
3. O `CMD` do Dockerfile é ignorado, pois o `entrypoint.sh` fixa o comando do Gunicorn.

**Melhorias:**
- Adicionar limite de tentativas e *timeout* na espera do banco.
- Executar na EC2 como serviço `systemd` (`Restart=always`) ou em container com política de reinício.
- Fazer o entrypoint respeitar o `CMD` (`exec "$@"`).

---

### X. Paridade entre desenvolvimento e produção — ⚠️ Atende parcialmente

**Princípio:** manter desenvolvimento, homologação e produção o mais semelhantes possível.

**Evidência (positivo):**
- O ambiente local usa PostgreSQL 16, o mesmo motor e a mesma versão principal do RDS (16.15), evitando a armadilha de testar em um banco e produzir em outro.
- O Python também coincide: 3.9 no container (`python:3.9-slim`) e 3.9.25 na EC2 (Amazon Linux 2023).

**Diferenças encontradas:**
1. Localmente a aplicação roda em **Docker** (Debian); na EC2, roda direto com **venv** (Amazon Linux 2023). Sistema operacional, pacotes de sistema e forma de execução diferem.
2. O compose monta `.:/app` como volume (conveniência de desenvolvimento) e usa credenciais fixas.
3. O banco local é um container; o de produção é um serviço gerenciado, com outros parâmetros e SSL.

**Melhorias:**
- Rodar o mesmo container Docker na EC2, o que aproxima os dois ambientes.
- Manter a mesma versão principal do PostgreSQL no compose e no RDS ao atualizar qualquer um deles.
- Evitar volume de código fora do ambiente de desenvolvimento.

---

### XI. Logs — ⚠️ Atende parcialmente

**Princípio:** tratar logs como fluxo de eventos, enviados para stdout/stderr.

**Evidência (positivo):**
- A aplicação não grava arquivos de log; tudo sai em stdout/stderr.
- O Dockerfile define `PYTHONUNBUFFERED=1`, de modo que as saídas aparecem sem atraso.

**Problemas:**
1. **Exceções não são registradas.** Os blocos `except` devolvem o erro na resposta HTTP, mas não escrevem nada no log. Se uma rota falhar, não há rastro no servidor.
2. O Gunicorn não registra requisições por padrão (o *access log* vem desativado).
3. A aplicação usa `print()` em vez do módulo `logging`, sem níveis nem formato estruturado.
4. O erro devolvido ao cliente inclui `str(e)`, que pode vazar detalhes internos.

**Melhorias:**
- Usar `logging` com níveis (INFO, ERROR) e, de preferência, formato JSON.
- Registrar a exceção no servidor e devolver mensagem genérica ao cliente.
- Habilitar o log de acesso: `--access-logfile - --error-logfile -`.
- Na AWS, enviar os logs ao **CloudWatch Logs** (CloudWatch Agent na EC2 ou driver `awslogs` no Docker), com retenção definida.

---

### XII. Processos administrativos — ✅ Atende

**Princípio:** executar tarefas administrativas como processos avulsos, com o mesmo código e configuração.

**Evidência:** a criação da tabela é um comando da própria aplicação (`@app.cli.command("init-db")`), executado com `flask init-db`. Usa o mesmo código, as mesmas variáveis de ambiente e pode ser rodado manualmente a qualquer momento.

**Observações:**
- Por ser um comando avulso, ele pode ser executado manualmente a qualquer momento. No deploy direto na EC2 (sem o `entrypoint.sh`), basta rodar `flask init-db` uma vez para criar a tabela, que foi exatamente o procedimento usado na implantação e o comportamento esperado pelo fator.
- Ressalva de qualidade: o `entrypoint.sh` chama `flask init-db || true`, e a função `init_db()` captura as exceções e só imprime. Assim, **falhas na criação da tabela podem passar despercebidas** e a aplicação sobe mesmo sem o schema. Isso é uma questão de tratamento de erros (veja também o fator XI), e não do princípio do fator.
- Para evolução do schema, será necessária uma ferramenta de migrações (Alembic, Flyway), pois `CREATE TABLE IF NOT EXISTS` não altera tabelas existentes.

**Melhorias:**
- Fazer a inicialização falhar de forma visível (código de saída diferente de zero).
- Adotar uma ferramenta de migrações de schema.

---

## 3. Recomendações para produção

As melhorias abaixo são sugestões para evolução da aplicação. **Não fazem parte do escopo da Fase 1**, que mantém a aplicação como está.

1. **Logs (XI):** registrar exceções com `logging`, habilitar o log de acesso do Gunicorn e não devolver `str(e)` ao cliente.
2. **Configuração (III):** ler `DB_PORT` no código, validar todas as variáveis obrigatórias e usar `sslmode=verify-full` na conexão com o RDS.
3. **Segredos (III):** o Secrets Manager já é usado na Fase 1, com leitura manual na EC2. A evolução é automatizar a leitura (por exemplo, no serviço `systemd`) e tratar a rotação do segredo sem reinício manual.
4. **Execução (IX):** rodar a aplicação como serviço `systemd` ou em container com reinício automático, e fazer o `/health` verificar o banco.
5. **Entrega (V, X):** versionar imagens Docker em um registry e automatizar o deploy com CI/CD, rodando o mesmo container em desenvolvimento e produção.
6. **Escalabilidade (VIII):** definir `--workers` e adotar pool de conexões ou RDS Proxy.
7. **Schema (XII):** adotar uma ferramenta de migrações de banco de dados.

---

## 4. Conclusão

O ToggleMaster já nasce com uma base sólida em vários fatores: **código único em Git, dependências declaradas, banco tratado como recurso anexado, processos sem estado, vínculo de porta e tarefa administrativa como comando avulso**. Isso explica por que a aplicação foi movida do ambiente local para a AWS trocando apenas variáveis de ambiente, o que a implantação confirmou.

As lacunas concentram-se na **operação em produção**: o ciclo de build/release/run é manual (fator V), os logs não registram falhas (XI), a gestão de segredos melhorou com o Secrets Manager, mas ainda depende de um passo manual a cada sessão e de reinício após a rotação (III), e há pouca resiliência de processos e conexões (VIII e IX). São problemas esperados em um MVP, e a maioria tem correção simples e de baixo custo. Para esta fase, a aplicação é implantada como está; as recomendações da seção 3, como pipeline de entrega, automação da leitura de segredos e observabilidade, alinham-se à evolução prevista do projeto para um sistema distribuído e automatizado nas próximas fases.
