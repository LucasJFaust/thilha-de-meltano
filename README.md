# trilha-de-meltano

```markdown
# Resumo Executivo: Pipeline Meltano para Extração de Dados Northwind

Este documento fornece um resumo detalhado dos comandos executados e suas respectivas explicações, servindo como um guia prático para o `README.md` do seu projeto Meltano.

---

## Extração de Dados Northwind com Meltano

Este guia documenta o processo de configuração e execução de um pipeline ELT (Extract, Load, Transform) usando Meltano para extrair dados da tabela `suppliers` do banco de dados Northwind (PostgreSQL) para um formato JSONL.

### Contexto

O objetivo é extrair dados da tabela `public.suppliers` do banco de dados `northwind`, que está rodando em um contêiner Docker PostgreSQL. Enfrentamos desafios com a seleção de streams e erros de validação de esquema devido à extração acidental de tabelas de metadados (`information_schema`).

---

## 1. Configuração e Verificação do Banco de Dados PostgreSQL

Antes de usar o Meltano, precisamos garantir que o banco de dados PostgreSQL esteja em execução e acessível.

### 1.1. Inicialização do Banco de Dados (via Docker Compose)

Assume-se que você tenha um arquivo `docker-compose.yml` e o script `northwind.sql` configurados para iniciar o serviço PostgreSQL e popular o banco de dados.

```bash
cd ~/indicium/estudo/trilha-de/ingestao/meltano # Ou a pasta onde está seu docker-compose.yml
docker-compose up -d
```

> **Explicação:** Este comando inicia o contêiner PostgreSQL em segundo plano e, geralmente, executa o script `northwind.sql` para criar o banco de dados `northwind` e suas tabelas.

### 1.2. Conexão com o Banco de Dados

Para verificar a acessibilidade do banco e suas tabelas.

```bash
psql -h localhost -p 5432 -U northwind_user -d northwind
```

> **Explicação:** Conecta-se ao banco de dados `northwind` como o usuário `northwind_user` através da porta 5432. Você será solicitado a inserir a senha (`password`).

### 1.3. Listar Tabelas Existentes

Dentro do prompt `northwind=>` do `psql`.

```sql
\dt
```

> **Explicação:** Lista todas as tabelas no esquema `public`. Se o output terminar com `(END)`, significa que a lista é maior que a tela. Pressione a tecla `q` para sair do paginador e voltar ao prompt.

### 1.4. Listar Schemas Existentes

Dentro do prompt `northwind=>` do `psql`.

```sql
\dn
```

> **Explicação:** Confirma a existência do esquema `public`, onde as tabelas do Northwind foram criadas.

---

## 2. Configuração do Projeto Meltano

Agora que o banco está funcionando, vamos configurar o Meltano.

### 2.1. Inicialização do Projeto Meltano

Certifique-se de estar na pasta que contém a estrutura do seu projeto Meltano (`~/indicium/estudo/trilha-de/ingestao/meltano`).

```bash
meltano init demo
```

> **Explicação:** Cria um novo projeto Meltano na pasta `demo`. A partir deste ponto, **todos os comandos Meltano devem ser executados dentro da pasta `demo`**.

```bash
cd demo
```

### 2.2. Adicionar Extrator (`tap-postgres`) e Loader (`target-jsonl`)

```bash
meltano add extractor tap-postgres --variant meltanolabs
meltano add loader target-jsonl --variant andyh1203
```

> **Explicação:** Adiciona os plugins necessários ao seu projeto Meltano. O `tap-postgres` extrai dados do PostgreSQL, e o `target-jsonl` os carrega em arquivos JSONL.

### 2.3. Configurar `tap-postgres` no `meltano.yml`

O arquivo `meltano.yml` (localizado em `~/indicium/estudo/trilha-de/ingestao/meltano/demo/meltano.yml`) precisa ser editado para informar ao Meltano como se conectar ao banco e quais dados extrair.

Edite o seu `meltano.yml` para que a seção do `tap-postgres` fique como no exemplo abaixo. **ATENÇÃO À INDENTAÇÃO DO `select`!** Ele deve estar no mesmo nível de `config`.

```yaml
version: 1
default_environment: dev
project_id: 0199069f-a82c-7e5a-a73e-3536fe32cc16
environments:
- name: dev
- name: staging
- name: prod
plugins:
  extractors:
  - name: tap-postgres
    variant: meltanolabs
    pip_url: meltanolabs-tap-postgres
    config:
      sqlalchemy_url: postgresql://northwind_user:password@localhost:5432/northwind
      schema: public # Indica ao tap para focar no esquema 'public'
    select: # <--- IMPORTANTE: ESTA LINHA 'select' DEVE ESTAR ALINHADA COM 'config'
      - "public-suppliers.*" # Seleciona TODAS as colunas da tabela 'suppliers' no esquema 'public'
      - "!*" # Exclui TODAS as outras streams que não foram explicitamente selecionadas acima (incluindo o 'information_schema')
  loaders:
  - name: target-jsonl
    variant: andyh1203
    pip_url: target-jsonl
```

> **Explicação das alterações:**
> *   `sqlalchemy_url`: Define a string de conexão para que o `tap-postgres` saiba onde encontrar o banco de dados.
> *   `schema: public`: Configuração específica do `tap-postgres` para direcionar a descoberta inicial de schemas.
> *   **`select` (indentação correta):** Esta seção é processada pelo *core* do Meltano e define o catálogo de dados a ser extraído.
>     *   `public-suppliers.*`: Inclui todas as colunas (`.*`) da stream `suppliers` que pertence ao esquema `public`.
>     *   `!*`: Esta é uma regra de exclusão abrangente. Ela diz ao Meltano para **excluir qualquer stream que não tenha sido explicitamente incluída nas linhas anteriores**. Isso é crucial para evitar a extração de tabelas de metadados como as do `information_schema`, que podem causar erros de validação no loader.

---

## 3. Instalação e Execução do Pipeline

Com o `meltano.yml` configurado, podemos instalar os plugins e executar o pipeline.

### 3.1. Instalar e Aplicar Configurações dos Plugins

**Este comando deve ser executado sempre que você alterar as configurações de um plugin no `meltano.yml`.**

```bash
meltano install extractor tap-postgres
```

> **Explicação:** Garante que o `tap-postgres` esteja instalado e que suas configurações mais recentes do `meltano.yml` sejam aplicadas.

### 3.2. Verificar Seleção de Streams (Opcional, para debug)

```bash
meltano select tap-postgres --list
```

> **Explicação:** Este comando lista todas as streams que o `tap-postgres` consegue descobrir no banco de dados e o status de seleção do Meltano (`[selected]`, `[excluded]`, etc.). O importante é que `public-suppliers` esteja `[selected]` e que as streams do `information_schema` estejam `[excluded]` ou simplesmente não apareçam.

### 3.3. Executar o Pipeline ELT

```bash
meltano run tap-postgres target-jsonl
```

> **Explicação:** Inicia o processo de extração (E) e carregamento (L). O `tap-postgres` se conecta ao PostgreSQL, extrai os dados da tabela `public.suppliers` (e somente dela, devido à configuração `select`) e envia esses dados para o `target-jsonl`, que os salva em arquivos JSONL.

---

## 4. Verificação dos Dados Extraídos

Após a execução bem-sucedida do pipeline, os dados estarão na pasta `output/` dentro do seu projeto `demo`.

```bash
ls output/
```

> **Explicação:** Lista os arquivos gerados. Você deve ver um arquivo como `public_suppliers.jsonl`.

```bash
cat output/public_suppliers.jsonl | head -n 5
```

> **Explicação:** Exibe as primeiras 5 linhas do arquivo JSONL, permitindo que você verifique se os dados da tabela `suppliers` foram extraídos corretamente.
```