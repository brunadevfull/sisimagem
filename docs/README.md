# Documentação do SisImagem

Sistema de Gestão de Documentos PAPEM-41/42 da PAPEM (instituição governamental). Este material foi escrito por mim para registrar o que existe hoje e orientar a manutenção. Sou a única programadora e conto com duas pessoas de apoio na operação.

---

## 📋 Estrutura da Documentação

```
docs/
├── README.md (este arquivo)
│
├── 01-analise-sistema-atual/
│   ├── visao-geral.md
│   ├── arquitetura-atual.md
│   └── analise-tecnica.md
│
├── 02-planejamento-migracao/
│   ├── estrategia-migracao.md
│   ├── comparativo-tecnologias.md
│   └── cronograma.md
│
└── 03-migracao-banco-dados/
    ├── oracle-para-postgresql.md
    └── scripts-migracao/
```

---

## 🎯 Documentos por Categoria

### 1️⃣ Análise do Sistema Atual

| Documento | Descrição | Status |
|-----------|-----------|--------|
| [Visão Geral](file:///home/bruna/sisimagem/SYSTEM_OVERVIEW.md) | Visão geral do sistema, stack e módulos | ✅ |
| [Arquitetura Atual](file:///home/bruna/sisimagem/docs/reverse-engineering-report.md) | Análise de engenharia reversa | ✅ |
| [Guia de Documentação](file:///home/bruna/sisimagem/docs/guia-documentacao-sisimagem.md) | Guia para documentação e reengenharia | ✅ |

**Resumo**: Sistema Java 8 com Servlets/JSP, Oracle/TRIM, jQuery 1.4.2

---

### 2️⃣ Planejamento de Migração

| Documento | Descrição | Status |
|-----------|-----------|--------|
| [Análise de Gaps](file:///home/bruna/sisimagem/docs/gaps-documentacao-migracao.md) | Lacunas na documentação atual | ✅ |
| [Plano de Ação](file:///home/bruna/sisimagem/docs/plano-acao-documentacao.md) | Plano para completar documentação | ✅ |
| [Comparativo de Tecnologias](file:///home/bruna/sisimagem/docs/migracao/comparativo-tecnologias-detalhado.md) | Análise técnica de opções | ✅ |

**Decisão**: Node.js + TypeScript + PostgreSQL + Next.js

---

### 3️⃣ Migração de Banco de Dados

| Documento | Descrição | Status |
|-----------|-----------|--------|
| [Plano Oracle → PostgreSQL](file:///home/bruna/sisimagem/docs/migracao/plano-migracao-oracle-postgresql.md) | Plano detalhado de migração | ✅ |

**Crítico**: Migração de Oracle para PostgreSQL é o componente mais complexo

---

## 🚀 Stack Tecnológica Recomendada

### Backend
- **Runtime**: Node.js 20 LTS
- **Framework**: Express.js ou Fastify
- **Linguagem**: TypeScript 5.x
- **ORM**: Prisma 5.x
- **Banco de Dados**: PostgreSQL 15

### Frontend
- **Framework**: Next.js 14
- **Linguagem**: TypeScript 5.x
- **UI**: Tailwind CSS + shadcn/ui
- **Estado**: React Query + Zustand

### Infraestrutura
- **SO**: Ubuntu Server 22.04 LTS
- **Proxy**: Nginx
- **Process Manager**: PM2
- **Cache**: Redis 7

---

## 📊 Benefícios da Migração

### Técnicos
- ✅ **Performance superior** para I/O intensivo
- ✅ **Segurança moderna** (elimina SQL Injection)
- ✅ **Interface responsiva** e acessível
- ✅ **Manutenibilidade** muito melhor
- ✅ **Type safety** com TypeScript
- ✅ **Escalabilidade** horizontal

---

## 📅 Cronograma Geral

### Fase 1: Preparação (4 semanas)
- Análise completa do sistema atual
- Extração do schema Oracle
- Setup de ambientes

### Fase 2: Migração de Banco (6 semanas) ⚠️ CRÍTICO
- Conversão Oracle → PostgreSQL
- Migração de dados
- Validação e testes

### Fase 3: Backend (10 semanas)
- APIs REST com Node.js
- Autenticação e autorização
- Lógica de negócio

### Fase 4: Frontend (10 semanas)
- Interface moderna com Next.js
- Responsividade mobile
- UX aprimorada

### Fase 5: Integração e Testes (4 semanas)
- Testes E2E
- Testes de carga
- Correção de bugs

### Fase 6: Deploy (2 semanas)
- Configuração de produção
- Migração final
- Go-live

**Total: 36 semanas (~9 meses)**

---

## ⚠️ Riscos Principais

| Risco | Probabilidade | Impacto | Mitigação |
|-------|---------------|---------|-----------|
| Perda de dados na migração | Baixa | Crítico | Backup completo + validação rigorosa |
| Incompatibilidades Oracle/PostgreSQL | Alta | Alto | Testes extensivos + conversão cuidadosa |
| Performance inferior | Média | Alto | Tuning + índices + cache |
| Downtime prolongado | Média | Alto | Migração incremental + rollback |
| Bugs em produção | Média | Alto | Testes E2E + monitoramento |

---

## 🎯 Próximos Passos

### Imediatos
1. ✅ Aprovar stack tecnológica (Node.js + PostgreSQL)
2. ⏳ Obter acesso ao Oracle (read-only)
3. ⏳ Iniciar análise detalhada do banco

### Curto Prazo
5. ⏳ Extração completa do schema Oracle
6. ⏳ Setup de ambientes (dev, staging, prod)
7. ⏳ Início da migração de banco

### Médio Prazo
8. ⏳ Desenvolvimento do backend
9. ⏳ Desenvolvimento do frontend
10. ⏳ Testes e validação

---

## 📖 Como Uso Esta Documentação

1. Leio este README para manter a visão geral e o estado dos documentos.
2. Consulto o [Guia de Documentação](file:///home/bruna/sisimagem/docs/guia-documentacao-sisimagem.md) sempre que preciso mapear fluxos ou regras.
3. Acesso o [Plano de Migração Oracle → PostgreSQL](file:///home/bruna/sisimagem/docs/migracao/plano-migracao-oracle-postgresql.md) quando estou avaliando a migração do banco.
4. Uso o [Comparativo de Tecnologias](file:///home/bruna/sisimagem/docs/migracao/comparativo-tecnologias-detalhado.md) apenas como referência técnica.

---

## 🔗 Links Importantes

### Documentação Técnica
- [PostgreSQL 15](https://www.postgresql.org/docs/15/)
- [Node.js 20](https://nodejs.org/docs/latest-v20.x/api/)
- [Next.js 14](https://nextjs.org/docs)
- [Prisma ORM](https://www.prisma.io/docs)
- [TypeScript](https://www.typescriptlang.org/docs/)

### Ferramentas de Migração
- [ora2pg](https://ora2pg.darold.net/) - Migração Oracle → PostgreSQL
- [pgloader](https://pgloader.io/) - Carregamento de dados
- [PostgreSQL Migration Tools](https://wiki.postgresql.org/wiki/Converting_from_other_Databases_to_PostgreSQL)

---

## 📞 Suporte

Para dúvidas sobre a documentação:
- **Repositório**: `/home/bruna/sisimagem`
- **Documentação**: `/home/bruna/sisimagem/docs`

---

**Última atualização**: 2025-12-20
