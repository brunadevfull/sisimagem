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

Desenhado a partir do inventário real de uso em `src/main/java/model/DAOTrim.java` (SQL efetivamente executado pela aplicação) e **confirmado contra o DDL real do Oracle/TRIM de homologação** (`DBMS_METADATA.GET_DDL`, obtido em 2026-09-08) — não mais especulativo.

O dump real confirma duas coisas importantes:

1. **As colunas reconstruídas a partir do código batem exatamente com o schema real**: `TSRECORD` tem de fato `RECORDID`, `TITLE`, `RCSTRUCTUREDTITLE`, `FULLRECORDID`, `RCCONTAINERURI`, `REGDATETIME`, `CREATIONDATETIME`, `RCRECTYPEURI`, `RCSCHEDULEURI`; `TSLOCATION` tem `LCNAME`, `LCIDNUMBER`, `LCJURGROUP`, `LCVALIDFROM`, `LCVALIDTO`; `TSRECELEC` tem `RESID`, `REFILENAME`, `REEXTENSION`, `RENAMEURI`, `REMODIFIEDDATETIME`; `TSRECLOC` tem `RLRECURI`, `RLLOCURI`, `RLDESCRIPTION`, `RLFROMDATETIME`, `RLTODATETIME`. Os relacionamentos inferidos por JOIN no `DAOTrim.java` (seção anterior) são FKs reais no Oracle: `FK_RCCONTAINERURI` (auto-relação em `TSRECORD`), `FK_RENAMEURI` (`TSRECELEC` → `TSLOCATION`), `FK_RLRECURI`/`FK_RLLOCURI` (`TSRECLOC` → `TSRECORD`/`TSLOCATION`), `FK_EVFIELDURI` (`TSEXFIELDV` → `TSEXFIELD`), `FK_LCJURGROUP` (`TSLOCATION` → `TSJURGROUP`).
2. **O dump inteiro tem mais de 100 tabelas** (agenda/reuniões, workflow, thesaurus, classificação de segurança multinível, código de barras, gestão de espaço físico de arquivo morto) — o SisImagem usa menos de 10 delas. Isso reforça a decisão da seção 2: o TRIM é um produto de gestão de registros corporativo completo, e abandoná-lo não perde funcionalidade real usada pelo SisImagem hoje.

Campos `NOT NULL`/nulináveis relevantes confirmados no Oracle real (informam os `NOT NULL` do DDL Postgres abaixo): em `TSRECORD`, apenas `URI`, `RECORDID`, `TITLE`, `RCSCHEDULEURI`, `FULLRECORDID`, `RCCONTAINERURI`, `RCSTRUCTUREDTITLE`, `REGDATETIME`, `CREATIONDATETIME` são obrigatórios — `RCRECTYPEURI` é nulável no schema real (usado sempre nos INSERTs do `DAOTrim.java`, mas o banco não obriga). Em `TSLOCATION`, só `URI` e `LCNAME` são obrigatórios — `LCIDNUMBER` (hash de senha) e `LCJURGROUP` são nuláveis (compatível com "locations" que não são usuários de login). Em `TSRECELEC`, só `URI`, `RENAMEURI` e `REMODIFIEDDATETIME` são obrigatórios — `RESID`/`REFILENAME`/`REEXTENSION` são nuláveis no Oracle real.

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

### 3.3 O que ainda falta (schema já confirmado, dados ainda não)

- Nomes/tipos exatos dos ~15 campos customizados de `documentos` — o DDL confirma a estrutura de `TSEXFIELD` (catálogo de campos), mas não os *valores* de `EXFIELDNAME` cadastrados nesta instância. Falta rodar a query da seção 4.1.
- Se `tramitacoes` é de fato um conceito de negócio usado ativamente (múltiplas tramitações por documento) ou só um registro de auditoria de criação — o código só mostra um insert por documento, nunca update/consulta de histórico de tramitação.
- Volume de dados real (linhas por tabela) — decide completo vs. incremental na Fase 4. Falta rodar a query da seção 4.2.

---

## 4. Queries restantes contra o Oracle de homologação

O DDL estrutural (schema, tipos, constraints, FKs) **já foi obtido** — a seção 3 acima já reflete o schema real. Faltam apenas duas consultas de dados, mais simples que extrair DDL:

### 4.1 Dicionário real dos campos customizados (resolve a lacuna 3.3)

```sql
SELECT URI, EXFIELDNAME FROM TSEXFIELD ORDER BY EXFIELDNAME;
```

### 4.2 Volume de dados (resolve a lacuna "Volume Estimado: A definir" dos planos anteriores)

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
| **0 — Destravar** | ~~Obter DDL real do Oracle~~ concluído. Restam as 2 queries de dados da seção 4 (campos customizados, volume) | Nenhum — acionável imediatamente |
| **1 — Schema e setup Postgres** | Fechar DDL da seção 3 com o resultado da seção 4; setup PostgreSQL 15 (`pg_trgm`, `unaccent` para busca) | Fase 0 |
| **2 — Backend** | Laravel 11 + Eloquent, portar regras de `DAOTrim`/`Operacao*` (autenticação, geração de numeração via `contadores_numeracao`, upload) | Fase 1 |
| **3 — Frontend** | Blade + Livewire dentro do próprio Laravel (confirmado) — sem API/SPA separada, sem overhead de token/CORS para o time atual (1 programadora + 2 apoio) | Fase 2 em paralelo |
| **4 — Migração de dados** | Migração completa do histórico para o Postgres novo (corte definitivo, sem manter o TRIM em paralelo); escolha de downtime completo vs. incremental conforme volume (Fase 0) | Fase 1 |
| **5 — Testes e go-live** | Checklist funcional, carga, segurança (queries parametrizadas, reset de senha); desligar acesso do SisImagem ao TRIM | Fases 2-4 |

---

## 6. Referências

- Inventário técnico completo de `DAOTrim.java` (tabelas, métodos, lógica de numeração, autenticação) — reconstruído nesta rodada de planejamento a partir do código real.
- [Gaps de Documentação](../01-analise-sistema-atual/gaps-documentacao-migracao.md) — checklist de lacunas, ainda válido para UI/regras de negócio fora do escopo de banco.
- [Plano Oracle → PostgreSQL (anterior)](../03-migracao-banco-dados/plano-migracao-oracle-postgresql.md) — checklist operacional de fases (backup, `pgbench`, monitoramento) continua útil; schema de exemplo nele está superado por este documento.
- [Recomendações de Stack (anterior)](./recomendacoes-stack-arquitetura.md) — superado pela decisão PHP/Laravel desta rodada.
