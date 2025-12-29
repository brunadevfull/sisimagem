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

## ✅ Documentos Essenciais

- [Visão Geral](file:///home/bruna/sisimagem/SYSTEM_OVERVIEW.md)
- [Arquitetura Atual](file:///home/bruna/sisimagem/docs/reverse-engineering-report.md)
- [Guia de Documentação](file:///home/bruna/sisimagem/docs/guia-documentacao-sisimagem.md)
- [Recomendações de Stack e Arquitetura](file:///home/bruna/sisimagem/docs/recomendacoes-stack-arquitetura.md)

**Resumo rápido**: Sistema Java 8 com Servlets/JSP, Oracle/TRIM, jQuery 1.4.2.

---

## 📌 Documentação de Apoio (usar só quando necessário)

- [Análise de Gaps](file:///home/bruna/sisimagem/docs/gaps-documentacao-migracao.md)
- [Plano de Ação](file:///home/bruna/sisimagem/docs/plano-acao-documentacao.md)
- [Comparativo de Tecnologias](file:///home/bruna/sisimagem/docs/migracao/comparativo-tecnologias-detalhado.md)
- [Plano Oracle → PostgreSQL](file:///home/bruna/sisimagem/docs/migracao/plano-migracao-oracle-postgresql.md)

---

## ✅ Checklist: A documentação atual está suficiente?

- [ ] Fluxos principais estão descritos (login, pesquisa, inclusão, anexos).
- [ ] Tabelas TRIM citadas nas operações estão mapeadas.
- [ ] Regras de validação e bloqueio estão registradas.
- [ ] Campos obrigatórios por tipo de documento estão descritos.
- [ ] Integrações externas (scanner, arquivos, TRIM) estão claras.

Se algum item acima ficar pendente, sigo o checklist abaixo.

---

## 🧭 Checklist de levantamento (se faltar informação)

- [ ] Revisar `ServletControlador.init()` e mapear `cmd` → `Operacao`.
- [ ] Listar campos e validações de cada JSP usada no fluxo principal.
- [ ] Extrair regras de negócio de `DAOTrim` e `Operacao*`.
- [ ] Mapear tabelas TRIM tocadas por cada operação.
- [ ] Registrar regras de senha (login + cadastro).
- [ ] Documentar integrações externas e caminhos de arquivo.

---

## 🧱 Checklist de decisões técnicas (stack ainda em avaliação)

- [ ] Definir critérios (segurança, manutenção, curva de aprendizado, integrações).
- [ ] Comparar opções de backend, frontend e banco com base nos critérios.
- [ ] Validar impacto de migração de Oracle → outro banco (queries e dados).
- [ ] Decidir se reimplementação será incremental ou big-bang.
- [ ] Registrar decisão final e justificativas.

---

## 🧩 Boas práticas para escolher arquitetura

- [ ] Manter o front controller (ou equivalente) para preservar o fluxo atual.
- [ ] Separar camada de domínio, persistência e transporte (controllers/handlers).
- [ ] Tratar autenticação e autorização como módulo isolado.
- [ ] Preferir contratos claros (DTOs) entre camadas.
- [ ] Priorizar migração do banco com testes de dados repetíveis.

---

## 📖 Como uso esta documentação

1. Leio os documentos essenciais para entender o fluxo atual.
2. Se faltar algo, sigo o checklist de levantamento.
3. Enquanto a stack estiver em avaliação, mantenho decisões e critérios no checklist técnico.

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
