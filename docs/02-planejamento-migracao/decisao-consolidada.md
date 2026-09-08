# Decisão Consolidada: Migração Oracle/TRIM → PostgreSQL + Reescrita do SisImagem

> **Status**: decisões fixadas em 2026-09-08. Este documento é a fonte de verdade para stack, arquitetura e schema de dados. Os documentos anteriores em `02-planejamento-migracao/` e `03-migracao-banco-dados/` mantêm valor como checklist/histórico, mas onde divergirem deste documento, **este prevalece**.

## 1. Por que este documento existe

O planejamento anterior travou porque três decisões de fundo nunca foram fechadas, e documentos diferentes assumiam respostas diferentes sem declarar isso:

1. Se o produto TRIM (Micro Focus Content Manager, ex-HP TRIM) seria abandonado ou mantido.
2. O schema Oracle documentado nos planos (`CREATE TABLE TSRECORD...`) era especulativo — não batia com as colunas reais usadas em `DAOTrim.java`.
3. Stack de aplicação contraditória entre documentos (Node/TS/Prisma vs. Java/Spring).

## 2. Decisões fixadas

| Decisão | Resolução | Justificativa |
|---|---|---|
| **Destino do TRIM** | **Abandonar** | `DAOTrim.java` já ignora completamente a camada de negócio do TRIM — faz SQL cru direto nas tabelas internas (`TSRECORD`, `TSLOCATION` etc.), sem usar workflow, retenção ou auditoria nativos do produto. O custo de abandoná-lo é baixo porque ele já não é aproveitado como produto, só como schema. TRIM não certifica PostgreSQL como backend — mantê-lo inviabilizaria tecnicamente o objetivo. |
| **Acesso ao Oracle** | Disponível (homologação) | Extração real do schema pode começar imediatamente (script na seção 4). Deixa de ser bloqueador teórico. |
| **Stack de aplicação** | **PHP 8.3 + Laravel 11** (confirmado) | Nenhum documento anterior cobria PHP. Laravel é o framework dominante no ecossistema PHP para este porte: Eloquent ORM, sistema de migrations nativo, Auth scaffolding, bom suporte a PostgreSQL. |
| **Escopo** | **Reescrita completa** | Novo backend (PHP/Laravel), novo frontend, novo banco (PostgreSQL) — substitui Java/JSP/Oracle-TRIM por completo. Não é troca de driver, é sistema novo. |

### Corte com o TRIM: sem dependência residual

Decisão fechada, sem pendência institucional: o SisImagem não precisa manter os documentos visíveis num TRIM corporativo, e o histórico será **migrado por completo** para o Postgres novo — sem manter o TRIM antigo como consulta paralela. Corte limpo: uma vez migrados os dados (seção 5, Fase 4), o SisImagem para de ler/escrever no schema TRIM definitivamente.

---

## 3. Schema PostgreSQL proposto

Desenhado a partir do inventário real de uso em `src/main/java/model/DAOTrim.java` (SQL efetivamente executado pela aplicação) — não dos exemplos especulativos dos planos anteriores, que não batiam com as colunas reais.

### 3.1 Decisões de modelagem (corrigindo problemas identificados no schema atual)

| Problema no TRIM/Oracle atual | Decisão no schema novo |
|---|---|
| Datas armazenadas como string `yyyyMMddHHmmss` | Colunas `timestamptz` reais |
| PKs geradas por `SELECT max(uri)+1` sem lock (race condition real) | `GENERATED ALWAYS AS IDENTITY` em todas as tabelas |
| Numeração de documento (`FULLRECORDID`) extraída via `substr()` de string, sem sequence real | Tabela dedicada `contadores_numeracao` com `SELECT ... FOR UPDATE` |
| Modelo EAV genérico (`TSEXFIELD`/`TSEXFIELDV`) para campos que na prática são fixos (`Documento.java` já usa getters/setters nomeados, não uso dinâmico) | **Achatar** os ~15 campos customizados em colunas nomeadas na tabela `documentos` |
| `TSLOCATION` sobrecarregada (usuário do sistema + responsável pelo documento + localização física) | Separar em `usuarios` (login/senha/grupo) e `grupos`; "responsável" vira FK para `usuarios`. **Pendente de confirmação de negócio**: se "localização física" (`TSRECLOC`/pasta) é um conceito distinto de usuário/grupo — o código sozinho não garante isso, precisa confirmar com quem opera o sistema |
| Senha com hash Whirlpool sem salt, comparada em Java, senha padrão hardcoded (`"marinha"`) | `bcrypt`/`argon2id` com salt. Migração de senha: reset forçado no primeiro login pós-corte (não dá para reidratar hash Whirlpool em bcrypt) |
| Vínculo indireto entre documento e arquivo físico (nome gerado por ano/mês do *upload*, não por `FULLRECORDID`; path ofuscado com `+` no lugar de `/`) | Coluna explícita `caminho_arquivo` em `anexos`, sem ofuscação, e path derivável do próprio ID do registro (mais robusto) |
| SQL 100% concatenado (sem bind variables) — SQL injection real hoje | Eloquent ORM / prepared statements em todo acesso a dados no novo backend |

### 3.2 DDL proposto

```sql
-- Grupos/perfis de acesso (substitui TSJURGROUP)
CREATE TABLE grupos (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nome        VARCHAR(100) NOT NULL UNIQUE,
    criado_em   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Usuários do sistema (substitui o uso de TSLOCATION como login)
CREATE TABLE usuarios (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    login               VARCHAR(100) NOT NULL UNIQUE,
    senha_hash          VARCHAR(255) NOT NULL, -- bcrypt/argon2id
    grupo_id            BIGINT NOT NULL REFERENCES grupos(id),
    bloqueado           BOOLEAN NOT NULL DEFAULT false,
    tentativas_invalidas SMALLINT NOT NULL DEFAULT 0,
    ultimo_acesso_em    TIMESTAMPTZ,
    criado_em           TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tipos de documento (substitui TSRECTYPE) — ex.: PAPEM-41, PAPEM-42, resposta PAPEM-42
CREATE TABLE tipos_documento (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    codigo      VARCHAR(20) NOT NULL UNIQUE,  -- ex.: 'PP41D', 'PP42D', 'PP41R', 'PP42R'
    nome        VARCHAR(150) NOT NULL,
    prefixo_numeracao VARCHAR(20) NOT NULL    -- formato do FULLRECORDID equivalente
);

-- Contador sequencial anual por tipo de documento (substitui a extração via substr(fullrecordid))
CREATE TABLE contadores_numeracao (
    tipo_documento_id BIGINT NOT NULL REFERENCES tipos_documento(id),
    ano               SMALLINT NOT NULL,
    ultimo_numero     INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (tipo_documento_id, ano)
);
-- Uso: SELECT ultimo_numero FROM contadores_numeracao WHERE tipo_documento_id = ? AND ano = ? FOR UPDATE;
--      UPDATE ... SET ultimo_numero = ultimo_numero + 1 WHERE ...;
--      (dentro da mesma transação do INSERT em documentos, eliminando a race condition atual)

-- Documentos (substitui TSRECORD, com campos EAV achatados de TSEXFIELD/TSEXFIELDV)
CREATE TABLE documentos (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    tipo_documento_id   BIGINT NOT NULL REFERENCES tipos_documento(id),
    documento_pai_id    BIGINT REFERENCES documentos(id), -- substitui RCCONTAINERURI (anexo -> documento pai)
    titulo              VARCHAR(500) NOT NULL,
    classificacao       VARCHAR(100),           -- substitui RCSTRUCTUREDTITLE
    numero_curto        VARCHAR(50) NOT NULL,   -- substitui RECORDID
    numero_completo     VARCHAR(100) NOT NULL UNIQUE, -- substitui FULLRECORDID
    data_documento       TIMESTAMPTZ NOT NULL,   -- substitui REGDATETIME
    responsavel_id      BIGINT NOT NULL REFERENCES usuarios(id),

    -- campos customizados achatados (nomes reais a confirmar/ajustar contra o dicionário de dados real do TRIM, seção 4)
    cpf                 VARCHAR(14),
    beneficiario        VARCHAR(255),
    protocolo           VARCHAR(50),
    urgente             BOOLEAN NOT NULL DEFAULT false,
    consignado          BOOLEAN,
    nip                 VARCHAR(50),
    -- ... demais campos customizados a extrair de TSEXFIELD (ver seção 4.2)

    criado_em           TIMESTAMPTZ NOT NULL DEFAULT now(),
    atualizado_em       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_documentos_tipo_ano ON documentos (tipo_documento_id, data_documento);
CREATE INDEX idx_documentos_pai ON documentos (documento_pai_id);
CREATE INDEX idx_documentos_cpf ON documentos (cpf);

-- Anexos / elemento eletrônico (substitui TSRECELEC)
CREATE TABLE anexos (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    documento_id        BIGINT NOT NULL REFERENCES documentos(id),
    nome_original       VARCHAR(255) NOT NULL,  -- substitui REFILENAME
    extensao            VARCHAR(20) NOT NULL,   -- substitui REEXTENSION
    caminho_arquivo     TEXT NOT NULL,          -- path explícito, sem ofuscação '+'/'/'
    enviado_por_id      BIGINT NOT NULL REFERENCES usuarios(id), -- substitui RENAMEURI
    modificado_em       TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Localização/tramitação do documento (substitui TSRECLOC) — pendente confirmar se é conceito
-- de negócio distinto de "responsável" (ver observação na seção 3.1)
CREATE TABLE tramitacoes (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    documento_id    BIGINT NOT NULL REFERENCES documentos(id),
    usuario_id      BIGINT NOT NULL REFERENCES usuarios(id),
    pasta           VARCHAR(255),               -- substitui RLDESCRIPTION
    a_partir_de     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.3 O que este DDL não resolve ainda

- Nomes/tipos exatos dos ~15 campos customizados de `documentos` — vieram parcialmente do glossário em `gaps-documentacao-migracao.md`, mas precisam ser confirmados contra `TSEXFIELD` real (query na seção 4.2).
- Se `tramitacoes` é de fato um conceito de negócio usado ativamente (múltiplas tramitações por documento) ou só um registro de auditoria de criação — o código só mostra um insert por documento, nunca update/consulta de histórico de tramitação.

---

## 4. Script de extração real do Oracle de homologação

Agora executável (acesso disponível). Roda contra o schema do TRIM em homologação para confirmar/corrigir o DDL da seção 3 antes de codar a migração de dados.

### 4.1 Estrutura das 7 tabelas confirmadas em uso

```sql
-- Rodar via sqlplus ou qualquer client Oracle, salvando cada resultado
SELECT table_name, num_rows, tablespace_name
FROM user_tables
WHERE table_name IN ('TSRECORD','TSEXFIELD','TSEXFIELDV','TSLOCATION','TSJURGROUP','TSRECLOC','TSRECELEC','TSRECTYPE')
ORDER BY table_name;

SELECT table_name, column_name, data_type, data_length, data_precision, data_scale, nullable, column_id
FROM user_tab_columns
WHERE table_name IN ('TSRECORD','TSEXFIELD','TSEXFIELDV','TSLOCATION','TSJURGROUP','TSRECLOC','TSRECELEC','TSRECTYPE')
ORDER BY table_name, column_id;

SELECT constraint_name, constraint_type, table_name, r_constraint_name, search_condition
FROM user_constraints
WHERE table_name IN ('TSRECORD','TSEXFIELD','TSEXFIELDV','TSLOCATION','TSJURGROUP','TSRECLOC','TSRECELEC','TSRECTYPE')
ORDER BY table_name;

SELECT index_name, table_name, uniqueness, column_name, column_position
FROM user_ind_columns
WHERE table_name IN ('TSRECORD','TSEXFIELD','TSEXFIELDV','TSLOCATION','TSJURGROUP','TSRECLOC','TSRECELEC','TSRECTYPE')
ORDER BY table_name, index_name, column_position;

-- DDL completo pronto (alternativa mais rápida às 3 queries acima)
SELECT DBMS_METADATA.GET_DDL('TABLE', table_name) FROM user_tables
WHERE table_name IN ('TSRECORD','TSEXFIELD','TSEXFIELDV','TSLOCATION','TSJURGROUP','TSRECLOC','TSRECELEC','TSRECTYPE');
```

### 4.2 Dicionário real dos campos customizados (resolve a lacuna da seção 3.3)

```sql
SELECT URI, EXFIELDNAME FROM TSEXFIELD ORDER BY EXFIELDNAME;
```

### 4.3 Volume de dados (resolve a lacuna "Volume Estimado: A definir" dos planos anteriores)

```sql
SELECT 'TSRECORD' tabela, COUNT(*) linhas FROM TSRECORD
UNION ALL SELECT 'TSEXFIELDV', COUNT(*) FROM TSEXFIELDV
UNION ALL SELECT 'TSRECELEC', COUNT(*) FROM TSRECELEC
UNION ALL SELECT 'TSRECLOC', COUNT(*) FROM TSRECLOC
UNION ALL SELECT 'TSLOCATION', COUNT(*) FROM TSLOCATION;
```

Esse número decide a estratégia de migração de dados (completa com downtime vs. incremental) — hoje nenhum documento anterior tinha esse dado.

---

## 5. Roadmap ajustado

Mantém o esqueleto de fases dos documentos anteriores (`plano-migracao-oracle-postgresql.md`, `comparativo-tecnologias-detalhado.md`), com a stack corrigida para PHP/Laravel e a Fase 0 tornada acionável agora:

| Fase | Conteúdo | Pré-requisito |
|---|---|---|
| **0 — Destravar** (agora) | Rodar script da seção 4 contra homologação | Nenhum — acionável imediatamente |
| **1 — Schema e setup Postgres** | Fechar DDL da seção 3 com dados reais da Fase 0; setup PostgreSQL 15 (`pg_trgm`, `unaccent` para busca) | Fase 0 |
| **2 — Backend** | Laravel 11 + Eloquent, portar regras de `DAOTrim`/`Operacao*` (autenticação, geração de numeração via `contadores_numeracao`, upload) | Fase 1 |
| **3 — Frontend** | A definir (Blade/Livewire dentro do próprio Laravel vs. SPA separada) — **decisão em aberto, não fechada nesta rodada** | Fase 2 em paralelo |
| **4 — Migração de dados** | Migração completa do histórico para o Postgres novo (corte definitivo, sem manter o TRIM em paralelo); escolha de downtime completo vs. incremental conforme volume (Fase 0) | Fase 1 |
| **5 — Testes e go-live** | Checklist funcional, carga, segurança (queries parametrizadas, reset de senha); desligar acesso do SisImagem ao TRIM | Fases 2-4 |

---

## 6. Referências

- Inventário técnico completo de `DAOTrim.java` (tabelas, métodos, lógica de numeração, autenticação) — reconstruído nesta rodada de planejamento a partir do código real.
- [Gaps de Documentação](../01-analise-sistema-atual/gaps-documentacao-migracao.md) — checklist de lacunas, ainda válido para UI/regras de negócio fora do escopo de banco.
- [Plano Oracle → PostgreSQL (anterior)](../03-migracao-banco-dados/plano-migracao-oracle-postgresql.md) — checklist operacional de fases (backup, `pgbench`, monitoramento) continua útil; schema de exemplo nele está superado por este documento.
- [Recomendações de Stack (anterior)](./recomendacoes-stack-arquitetura.md) — superado pela decisão PHP/Laravel desta rodada.
