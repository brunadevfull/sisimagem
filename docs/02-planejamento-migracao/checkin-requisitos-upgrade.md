# Check-in de Requisitos para Upgrade do SisImagem

## Objetivo
Este check-in consolida os **requisitos funcionais e não funcionais** já identificados no sistema atual para apoiar o levantamento de requisitos do upgrade.

> Status usado neste documento:
> - **ATENDIDO (AS-IS)**: já existe no sistema atual.
> - **PARCIAL (AS-IS)**: existe, mas com lacunas de regra, robustez ou documentação.
> - **NÃO ATENDIDO / A DEFINIR (TO-BE)**: precisa ser definido para o upgrade.

---

## 1) Requisitos Funcionais (RF)

### RF-01 — Autenticação de usuário
- **Descrição**: usuário deve autenticar com login e senha para acessar o sistema.
- **Status**: **ATENDIDO (AS-IS)**.
- **Evidência no legado**: operação `login`, sessão com atributo `usuario`, filtro de acesso às views.
- **Critério para upgrade**:
  - manter autenticação centralizada;
  - registrar trilha mínima de tentativa e sucesso de login.

### RF-02 — Bloqueio por tentativas inválidas
- **Descrição**: bloquear usuário após tentativas consecutivas de senha inválida.
- **Status**: **ATENDIDO (AS-IS)** (bloqueio na 5ª tentativa).
- **Risco no legado**: contagem de tentativas está em sessão web, podendo variar por sessão/navegador.
- **Critério para upgrade**:
  - contabilização persistente por usuário;
  - política de desbloqueio parametrizável.

### RF-03 — Troca obrigatória de senha padrão
- **Descrição**: forçar alteração de senha quando usuário estiver com senha padrão.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - manter regra;
  - remover senha padrão fixa em texto claro do código.

### RF-04 — Pesquisa de documentos
- **Descrição**: pesquisar documentos e exibir resultado com dados principais e detalhamento.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - manter filtros essenciais do negócio;
  - permitir paginação e ordenação consistente.

### RF-05 — Detalhamento de documento
- **Descrição**: abrir detalhes do documento selecionado com metadados e anexos.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - manter visualização completa de metadados;
  - melhorar legibilidade e rastreabilidade de campos.

### RF-06 — Inclusão de documento
- **Descrição**: cadastrar documento com metadados e arquivo no repositório.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - validação de tipo/tamanho de arquivo;
  - feedback transacional claro (sucesso/erro).

### RF-07 — Inclusão de anexos (incluindo PAPEM-41)
- **Descrição**: anexar arquivo a documento já existente, incluindo fluxo PAPEM-41.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - manter vínculo com documento pai;
  - garantir consistência de nomenclatura e versionamento.

### RF-08 — Fluxos PAPEM-42 (documento e resposta)
- **Descrição**: suportar inclusão de documento PAPEM-42 e resposta PAPEM-42.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - explicitar regras de negócio por tipo de fluxo;
  - padronizar obrigatoriedade de campos por tipo documental.

### RF-09 — Geração de próximo RecordId
- **Descrição**: calcular próximo identificador com base no tipo documental e ano.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - evitar colisão em cenários concorrentes;
  - definir estratégia transacional (sequence/lock/serviço único).

### RF-10 — Gestão de usuários
- **Descrição**: incluir, alterar, detalhar, pesquisar e excluir usuários.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - definir perfis e permissões por ação;
  - trilha de auditoria para ações administrativas.

### RF-11 — Encerramento de sessão
- **Descrição**: encerrar sessão de usuário explicitamente.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - invalidar tokens/sessões de forma consistente;
  - redirecionamento previsível para login.

### RF-12 — Localização/abertura de arquivo digital
- **Descrição**: localizar e abrir arquivo associado ao registro.
- **Status**: **ATENDIDO (AS-IS)**.
- **Critério para upgrade**:
  - controlar autorização de acesso por perfil;
  - padronizar resposta para arquivo não encontrado.

### RF-13 — Integração de digitalização (scanner)
- **Descrição**: acionar fluxo de digitalização de documentos.
- **Status**: **PARCIAL (AS-IS)**.
- **Observação**: dependência de integração legada com executável local.
- **Critério para upgrade**:
  - definir estratégia moderna de digitalização (serviço local, API ou estação dedicada);
  - desacoplar dependência de componente desktop legado.

---

## 2) Requisitos Não Funcionais (RNF)

### RNF-01 — Segurança de autenticação e credenciais
- **Status atual**: **PARCIAL (AS-IS)**.
- **Achados**:
  - existe hash de senha e bloqueio por tentativas;
  - credenciais e parâmetros sensíveis ainda aparecem acoplados em código/configurações legadas.
- **Meta de upgrade**:
  - gestão de segredos por cofre/variáveis seguras;
  - política formal de senha e sessão.

### RNF-02 — Segurança de transporte (HTTPS)
- **Status atual**: **PARCIAL (AS-IS)** (há indício de configuração comentada).
- **Meta de upgrade**:
  - HTTPS obrigatório ponta a ponta;
  - HSTS e cookies seguros.

### RNF-03 — Controle de acesso/autorização
- **Status atual**: **PARCIAL (AS-IS)**.
- **Achados**: há controle de sessão, mas matriz de autorização por perfil precisa ser formalizada.
- **Meta de upgrade**:
  - RBAC com políticas por recurso/ação;
  - negação por padrão.

### RNF-04 — Auditabilidade e rastreabilidade
- **Status atual**: **PARCIAL (AS-IS)**.
- **Meta de upgrade**:
  - trilha auditável para login, consulta, inclusão, alteração e exclusão;
  - correlação por ID de requisição.

### RNF-05 — Desempenho
- **Status atual**: **PARCIAL (AS-IS)**.
- **Achados**: consultas SQL extensas e sem camada de otimização explícita no legado.
- **Meta de upgrade**:
  - metas de tempo de resposta por operação;
  - paginação, índices e observabilidade de queries.

### RNF-06 — Escalabilidade
- **Status atual**: **NÃO ATENDIDO / A DEFINIR (TO-BE)**.
- **Meta de upgrade**:
  - arquitetura escalável horizontalmente para camadas web/API;
  - separação clara entre aplicação e armazenamento de arquivos.

### RNF-07 — Confiabilidade e resiliência
- **Status atual**: **PARCIAL (AS-IS)**.
- **Meta de upgrade**:
  - política de retry controlado;
  - tratamento padronizado de falhas de banco, arquivo e scanner.

### RNF-08 — Manutenibilidade
- **Status atual**: **PARCIAL (AS-IS)**.
- **Achados**: regras distribuídas entre servlet/operações/DAO e pouca padronização de camadas.
- **Meta de upgrade**:
  - modularização por domínio;
  - testes automatizados e padrões de código.

### RNF-09 — Portabilidade e independência de estação
- **Status atual**: **PARCIAL (AS-IS)**.
- **Meta de upgrade**:
  - remover dependências locais específicas de desktop para funcionalidades críticas.

### RNF-10 — Observabilidade operacional
- **Status atual**: **PARCIAL (AS-IS)**.
- **Meta de upgrade**:
  - logs estruturados;
  - métricas e alertas (erro, latência, disponibilidade).

### RNF-11 — Conformidade e proteção de dados
- **Status atual**: **NÃO ATENDIDO / A DEFINIR (TO-BE)**.
- **Meta de upgrade**:
  - mapear dados pessoais/sensíveis;
  - definir retenção, descarte e controles de acesso aderentes à política institucional/LGPD.

### RNF-12 — Usabilidade e acessibilidade
- **Status atual**: **PARCIAL (AS-IS)**.
- **Meta de upgrade**:
  - interface responsiva;
  - padrões mínimos de acessibilidade (teclado, contraste, feedback de erro).

---

## 3) Priorização sugerida para levantamento detalhado

### Onda 1 (crítico)
1. Segurança (RNF-01, RNF-02, RNF-03, RNF-11).
2. Fluxos centrais de negócio (RF-01 a RF-09).
3. Gestão de usuários e auditoria (RF-10, RNF-04).

### Onda 2 (alta)
1. Desempenho e confiabilidade (RNF-05, RNF-07, RNF-10).
2. Scanner e portabilidade operacional (RF-13, RNF-09).

### Onda 3 (evolutivo)
1. Escalabilidade e modernização de arquitetura (RNF-06, RNF-08).
2. UX e acessibilidade (RNF-12).

---

## 4) Checklist de entrevista para fechar requisitos TO-BE

- [ ] Quais perfis de usuário existirão no upgrade e quais ações cada perfil poderá executar?
- [ ] Qual política de senha, bloqueio e desbloqueio será adotada?
- [ ] Quais filtros de pesquisa são obrigatórios e quais são opcionais?
- [ ] Quais formatos e tamanhos de arquivo serão permitidos por tipo documental?
- [ ] Qual será a estratégia oficial para digitalização (scanner) sem dependência de executável legado?
- [ ] Quais eventos precisam de trilha de auditoria obrigatória?
- [ ] Quais metas de desempenho (SLA/SLO) serão assumidas para pesquisa, inclusão e abertura de documentos?
- [ ] Quais requisitos de retenção e descarte documental precisam ser cumpridos?
- [ ] Quais integrações externas serão mandatórias na primeira versão do upgrade?

---

## 5) Resultado esperado deste check-in

Com este material, já é possível iniciar um **workshop de levantamento de requisitos TO-BE** com áreas usuárias e TI, reduzindo retrabalho e priorizando riscos de segurança, operação e continuidade do serviço.
