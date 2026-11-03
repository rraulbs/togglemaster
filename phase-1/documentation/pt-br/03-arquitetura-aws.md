# Arquitetura AWS do ToggleMaster (Fase 1)

Este documento descreve a arquitetura proposta para implantar a aplicação monolítica ToggleMaster na AWS e o que foi efetivamente provisionado. Na Fase 1, a aplicação é implantada **como está**: uma instância EC2 em sub-rede pública executa a API, um banco PostgreSQL gerenciado (RDS) fica em sub-rede privada e as credenciais do banco ficam no AWS Secrets Manager.

As seções 1 a 8 descrevem a arquitetura no estado final, conforme criada no console da AWS. Os ajustes feitos durante a implantação em relação ao planejamento inicial, com os motivos, estão na seção 9.

---

## 1. Visão geral

![Diagrama da arquitetura AWS do ToggleMaster](../../images/togglemaster_aws_architecture.png)

---

## 2. Decisões de projeto

1. **Duas sub-redes privadas.** O RDS exige um *DB subnet group* com sub-redes em pelo menos duas Zonas de Disponibilidade, mesmo em uma instância Single-AZ. A instância é criada em **uma** das sub-redes do grupo. Como o campo *Availability Zone* ficou em *No preference*, a AWS escolheu `us-east-1b`, e o banco está na sub-rede privada B. A sub-rede privada A fica como reserva do grupo.
2. **PostgreSQL 16 no RDS.** O suporte padrão do RDS PostgreSQL 13 terminou em 28/02/2026. A criação do banco com a 13.23 foi recusada pelo console, que exigiu o *Extended Support* (cobrança adicional). A versão escolhida foi a **16.15**, e o `docker-compose.yaml` local usa `postgres:16` para manter a paridade entre desenvolvimento e produção. A aplicação usa apenas SQL básico (`SERIAL`, `VARCHAR`, `BOOLEAN`, `TIMESTAMPTZ`), então é compatível com as duas versões.
3. **SSL no banco.** A partir do PostgreSQL 15, o RDS exige conexões com SSL por padrão (`rds.force_ssl=1`). A aplicação funcionou sem alterações, porque o `sslmode` padrão do `psycopg2` (`prefer`) negocia SSL. A validação do certificado não é feita nesta fase.
4. **Sem NAT Gateway.** O RDS não precisa acessar a internet, e a EC2 já tem saída pela rota para o Internet Gateway. O NAT teria custo fixo sem benefício.
5. **Credenciais no AWS Secrets Manager.** O RDS cria e mantém o segredo com o usuário e a senha do banco. A EC2 lê esse segredo por meio de uma IAM Role (`LabInstanceProfile`) e exporta as variáveis de ambiente antes de iniciar o Gunicorn. A senha não é digitada na instância nem escrita no código, e o `app.py` não precisa ser alterado, pois continua lendo `DB_USER` e `DB_PASSWORD` do ambiente.
6. **Amazon Linux 2023 na EC2.** O Python 3.9 da AMI é o mesmo da imagem `python:3.9-slim` do Dockerfile, o que aproxima o ambiente local do da EC2, e a AWS CLI já vem instalada (necessária para ler o segredo). O preço da instância é o mesmo do Ubuntu comum, então a escolha foi técnica, e não de custo.

---

## 3. Rede (VPC)

**Região:** `us-east-1` (N. Virginia). O AWS Academy limita o acesso a `us-east-1` e `us-west-2`, e a chave `vockey` só existe em `us-east-1`.

| Recurso | Configuração | Nome |
|---|---|---|
| VPC | `10.0.0.0/16`, DNS hostnames habilitado | `togglemaster-vpc` |
| Sub-rede pública | `10.0.1.0/24`, `us-east-1a` (EC2), IP público automático | `togglemaster-public-1a` |
| Sub-rede privada A | `10.0.2.0/24`, `us-east-1a` (reserva do subnet group) | `togglemaster-private-1a` |
| Sub-rede privada B | `10.0.3.0/24`, `us-east-1b` (RDS) | `togglemaster-private-1b` |
| Internet Gateway | Anexado à VPC | `togglemaster-igw` |
| Route table pública | `local` e `0.0.0.0/0 → IGW`, associada à sub-rede pública | `togglemaster-public-rt` |
| Route table privada | Apenas `local`, associada às duas sub-redes privadas | `togglemaster-private-rt` |

---

## 4. Security Groups

### 4.1 SG-EC2 (`togglemaster-sg-ec2`)

**Entrada:**

| Tipo | Porta | Origem | Motivo |
|---|---|---|---|
| SSH | 22 | `<IP do operador>/32` | Administração, apenas do IP do operador |
| TCP personalizado | 5000 | `0.0.0.0/0` | A API (Gunicorn) escuta na porta 5000 |

**Saída:** padrão (tudo liberado), necessária para `dnf`, `pip` e para a leitura do segredo no Secrets Manager (HTTPS).

### 4.2 SG-RDS (`togglemaster-sg-rds`)

**Entrada:**

| Tipo | Porta | Origem |
|---|---|---|
| PostgreSQL | 5432 | `togglemaster-sg-ec2` (referência ao Security Group, não a um CIDR) |

### 4.3 Observações

- **HTTP/HTTPS:** as regras das portas 80 e 443 chegaram a ser criadas, conforme o enunciado, mas foram removidas, porque nada escuta nelas: a aplicação serve HTTP diretamente na porta 5000. Liberar 80/443 só faria sentido com um proxy reverso ou Load Balancer, que ficam fora do escopo da Fase 1.
- **IP do operador:** com IP dinâmico, a regra de SSH precisa ser atualizada (opção *My IP*) quando o IP mudar.

---

## 5. EC2

| Item | Escolha | Motivo |
|---|---|---|
| AMI | Amazon Linux 2023 (kernel 6.1), x86_64 | Python 3.9 igual ao do Dockerfile e AWS CLI pré-instalada. Usuário de acesso: `ec2-user` |
| Tipo | `t3.micro` (2 vCPU, 1 GiB) | Suficiente para um Gunicorn com baixa carga; permitido no AWS Academy |
| Créditos de CPU | Standard | O modo *Unlimited* pode gerar cobrança extra |
| Disco | 8 GiB gp3 (padrão) | A aplicação é pequena |
| Rede | Sub-rede pública, IP público automático, SG-EC2 | Acesso externo |
| Chave SSH | `vockey` (Academy), arquivo `labsuser.pem` | Acesso via SSH |
| IAM Role | `LabInstanceProfile` (AWS Academy) | Permite à EC2 ler o segredo do banco sem credenciais fixas. Em uma conta pessoal, uma role com `secretsmanager:GetSecretValue` restrita ao segredo do RDS |
| Metadados | IMDSv2 obrigatório | Mais seguro |
| Ferramentas | AWS CLI 2.38 (pré-instalada), Python 3.9.25 | Leitura do segredo e execução da aplicação |
| Elastic IP | Não utilizado | O IP público dinâmico muda ao parar e iniciar a instância; a AWS cobra por endereços IPv4 públicos |

**Por que não Ubuntu.** O Ubuntu 24.04 traz Python 3.12, e o `psycopg2-binary==2.9.5` do `requirements.txt` é de 2022, o que representa risco de incompatibilidade. A 22.04 aparecia no seletor rápido do console apenas na variante com SQL Server (licença paga). O Amazon Linux 2023 evitou os dois problemas.

---

## 6. RDS

| Item | Escolha |
|---|---|
| Engine | PostgreSQL 16.15 (*Extended Support* desabilitado) |
| Template | Free tier |
| Implantação | Instância única, sem Multi-AZ |
| Classe | `db.t3.micro` |
| Armazenamento | gp3, 20 GiB, sem autoscaling, criptografado (`aws/rds`) |
| Identificador | `togglemaster-db` |
| Usuário master | `postgres` |
| Senha | Gerenciada pelo RDS no AWS Secrets Manager (*Managed in AWS Secrets Manager*) |
| Initial database name | `togglemaster` |
| Rede | `togglemaster-vpc`, subnet group `togglemaster-db-subnet-group` (`private-1a` e `private-1b`), AZ *No preference* (a AWS escolheu `us-east-1b`) |
| Public access | Não |
| Security Group | `togglemaster-sg-rds` |
| Backup | Retenção de 1 dia |
| Extras | Enhanced Monitoring e *Log exports* desligados; Database Insights no modo Standard (7 dias, sem custo); Deletion protection desligada para facilitar a limpeza |

O campo **Initial database name** é essencial: se ficar em branco, o RDS não cria o banco `togglemaster`, e a aplicação falha com `database "togglemaster" does not exist`.

---

## 7. Deploy na EC2

O código foi enviado da máquina local para a EC2 com `scp`, usando a mesma chave do SSH (sem colocar credenciais do GitHub na instância).

```powershell
# Na máquina local (PowerShell)
scp -i C:\Users\<usuario>\.ssh\labsuser.pem -r "<caminho>\toggle-master-monolith" ec2-user@<ip-publico>:~/
ssh -i C:\Users\<usuario>\.ssh\labsuser.pem ec2-user@<ip-publico>
```

```bash
# Na EC2
cd ~/toggle-master-monolith
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### Variáveis de ambiente

`DB_HOST`, `DB_PORT` e `DB_NAME` não são segredos e continuam como variáveis comuns. `DB_USER` e `DB_PASSWORD` vêm do segredo criado pelo RDS, que contém os campos `username` e `password`. A senha nunca é digitada, então não fica no histórico do shell nem em arquivo:

```bash
export SECRET_ARN='<arn-do-segredo>'
export DB_HOST='<endpoint-do-rds>'
export DB_PORT='5432'
export DB_NAME='togglemaster'
export SECRET_JSON=$(aws secretsmanager get-secret-value --secret-id "$SECRET_ARN" --region us-east-1 --query SecretString --output text)
export DB_USER=$(echo "$SECRET_JSON" | python3 -c "import sys,json; print(json.load(sys.stdin)['username'])")
export DB_PASSWORD=$(echo "$SECRET_JSON" | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")
unset SECRET_JSON
```

O `app.py` não lê `DB_PORT` e usa a porta padrão 5432 (ponto já registrado na análise 12-Factor).

### Criar a tabela e iniciar a aplicação

No Docker, o `entrypoint.sh` executa `flask init-db` automaticamente. Na EC2, rodando só o Gunicorn, esse passo precisa ser feito uma vez:

```bash
flask init-db                              # cria a tabela "flags" no RDS
gunicorn --bind 0.0.0.0:5000 app:app       # fica em primeiro plano
```

### Validação

De fora da EC2, pelo IP público:

```powershell
curl.exe http://<ip-publico>:5000/health
curl.exe -X POST -H "Content-Type: application/json" -d '{\"name\":\"new-feature\",\"is_enabled\":true}' http://<ip-publico>:5000/flags
curl.exe http://<ip-publico>:5000/flags
```

### Limitações

- As variáveis valem apenas para a sessão do terminal. Ao reconectar por SSH, é preciso exportá-las de novo e reiniciar o Gunicorn.
- A aplicação lê a senha uma única vez, ao iniciar. O segredo gerenciado pelo RDS é rotacionado automaticamente (por padrão, a cada 7 dias), e após uma rotação a aplicação precisa ser reiniciada para ler a senha nova.
- O Gunicorn roda em primeiro plano: se a sessão SSH fecha, a aplicação cai. Em produção, usaria `systemd` (fator IX do 12-Factor).

---

## 8. Considerações sobre o ambiente

- **AWS Academy (Learner Lab):** o IAM é restrito (usa-se a `LabRole`) e a sessão do lab dura 4 horas. Ao encerrar, EC2 e RDS são parados e o **IP público da EC2 muda**; o endpoint do RDS não muda. A `LabInstanceProfile` permitiu a leitura do segredo no Secrets Manager.
- **Conta pessoal:** contas criadas a partir de 15/07/2025 recebem créditos (até US$ 200) por 6 meses no plano gratuito, em vez dos 12 meses de 750 horas para `t2.micro`/`t3.micro` das contas antigas. As regras vigentes devem ser conferidas no painel Billing.
- **IPv4 público tem custo**, mesmo em uso (cerca de US$ 0,005 por hora). Esse valor entra na estimativa de custos.
- **Secrets Manager tem custo:** cerca de US$ 0,40 por segredo por mês, mais as chamadas à API. Esse valor também entra na estimativa de custos.
- **Tráfego entre AZs:** a EC2 (`us-east-1a`) acessa o RDS (`us-east-1b`). Funciona normalmente pela rota `local` da VPC, e o custo de transferência é desprezível neste volume.

---

## 9. Decisões que mudaram na implementação

| Tema | Planejado | Implementado | Motivo |
|---|---|---|---|
| PostgreSQL | 13 (paridade com o compose) | **16.15**, compose atualizado para `postgres:16` | A 13.23 saiu do suporte padrão e o console exigiu Extended Support (cobrança extra) |
| Sub-redes privadas | 1 | **2** | O DB subnet group exige 2 AZs |
| Posição do RDS | Privada A (AZ a) | **Privada B (`us-east-1b`)** | AZ *No preference*: a AWS escolheu a 1b |
| Sistema da EC2 | Ubuntu 22.04 | **Amazon Linux 2023** | Python 3.9 igual ao do Dockerfile; AWS CLI pré-instalada; a 22.04 só aparecia com SQL Server pago |
| Portas do SG-EC2 | 22, 80, 443 | **22 e 5000** | Nada escuta em 80/443; a API usa a 5000 |
| Credenciais | Variáveis de ambiente digitadas | **Secrets Manager** lido via IAM Role | Evita senha em histórico, arquivo ou código |
| Criação da tabela | Automática (entrypoint) | **`flask init-db` manual** | Na EC2 o Gunicorn é iniciado sem o `entrypoint.sh` |

---

## 10. Fora do escopo da Fase 1

Para manter a arquitetura simples, não fazem parte desta fase: NAT Gateway, Load Balancer, HTTPS (ACM), Multi-AZ, Auto Scaling, Elastic IP, `systemd` e CloudWatch Agent. Esses itens aparecem como evolução nas recomendações da análise 12-Factor.

Como a API não tem autenticação e a porta 5000 fica aberta ao público, recomenda-se manter o ambiente ligado apenas durante o desenvolvimento e a avaliação, removendo os recursos em seguida.