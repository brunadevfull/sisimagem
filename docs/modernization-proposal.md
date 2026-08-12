# Proposta de Modernização do SisImagem

## Comparativo resumido: sistema legado vs. proposta
- **Arquitetura**: legado em servlets/JSP acoplados a DAOs JDBC com conexões compartilhadas; proposta em camadas hexagonais com APIs REST, IoC e adaptadores para persistência e storage.
- **Tecnologia**: Java 1.4/JSP/jQuery 1.4.2 no legado; proposta sem Java priorizando FastAPI/Go ou NestJS, SPA em React/Next.js/Angular e ORM com pooling seguro.
- **Banco e armazenamento**: legado concentra metadados e arquivos no Oracle/TRIM e diretório de rede; proposta mantém leitura/escrita no Oracle via drivers oficiais durante a transição e adiciona PostgreSQL + object storage versionado, com opção complementar de MongoDB/OpenSearch para documentos.
- **Segurança e observabilidade**: legado sem hashing forte, logging mínimo e sem métricas; proposta traz OAuth2/OIDC, RBAC, validação de entrada, logs estruturados, métricas/tracing (OpenTelemetry) e hardening.
- **Experiência do usuário**: legado com JSP não responsivo; proposta com SPA responsiva, uploads com feedback, busca avançada e acessibilidade.

## 1. Modelo de negócio atendido
- **Gestão de documentos e anexos**: autenticação de usuários, pesquisa e detalhamento de registros, inclusão de documentos/anexos (fluxos PAPEM-41/42) e geração de identificadores sequenciais no repositório TRIM/Oracle e diretório de arquivos no servidor.
- **Fluxos operacionais**: upload/scan de documentos, controle de usuários e grupos, e fornecimento de arquivos para download/visualização conforme permissões.

## 2. Principais módulos/telas/funcionalidades
- **Servlet frontal e comandos (`controller`)**: `ServletControlador` roteia comandos via `cmd` para operações de login, gestão de usuários, busca/inclusão de documentos, geração de recordId, anexos e encerramento de sessão, usando redirecionamentos JSP. Serviços específicos cobrem download e escaneamento de documentos.
- **Persistência JDBC (`model/DAOTrim`)**: conexões Oracle/TRIM criadas manualmente; consultas para autenticação, busca de campos, geração de IDs e gravação de metadados/anexos.
- **Domínio (`bean`)**: objetos simples para Documento, Anexo, Usuário e Grupo, carregados pelo DAO e usados pelos comandos.
- **Web/UI (`src/main/webapp`)**: JSPs de login, pesquisa e inclusão de documentos, recursos estáticos (CSS, JS jQuery 1.4.2) e configurações `web.xml` com filtro de login.

## 3. Problemas técnicos e de arquitetura
- **Acoplamento e separação de responsabilidades**: o servlet frontal instância diretamente cada operação (`new OperacaoX`) e usa JSPs via forward, dificultando testes e evolução. Não há inversão de controle nem camadas claras para serviço/negócio.【F:src/main/java/controller/ServletControlador.java†L20-L130】
- **Persistência frágil**: JDBC manual com strings de conexão embutidas, comentários de ambientes e credenciais fixas, sem pooling ou gestão de transação; o `Connection` é guardado em atributo compartilhado, sem thread safety ou fechamento adequado, expondo vazamento de recursos.【F:src/main/java/model/DAOTrim.java†L34-L145】
- **Segurança**: ausência de hashing seguro de senha na autenticação e manipulação de sessão rudimentar; dependência de executável local para scanner via header pode abrir vetores de abuso. Não há validação consistente de entradas nem proteção contra CSRF/XSS.
- **UX/Front-end legado**: JSPs server-side com jQuery 1.4.2, ausência de design responsivo e experiência moderna; dependência de plugins de máscara desatualizados.
- **Manutenibilidade/Testabilidade**: lógica de negócio misturada em servlets/DAOs sem testes automatizados; ausência de DI, DTOs e camada de serviço torna alterações arriscadas.
- **Operação e observabilidade**: logging mínimo, sem métricas, tracing, health checks ou configuração externa para ambientes, dificultando monitoramento e operação em produção.

## 4. Nova solução proposta
### Arquitetura recomendada
- **Backend em camadas (hexagonal)**: controladores REST finos, camada de aplicação orquestrando casos de uso, domínio com regras de negócio e portas para persistência/serviços externos. Adaptadores para banco (ORM) e armazenamento de arquivos/object storage.
- **Serviços**: API monolito modular inicial (FastAPI/Go) com módulos de identidade/autorização, gestão documental, anexos e integração TRIM. Possibilidade futura de extrair microserviços (ex.: ingestão/scan) conforme necessidade de escala.
- **Front-end SPA**: aplicação React/Next.js (ou Angular) consumindo APIs, com roteamento protegido e componentes reutilizáveis para pesquisa, upload e visualização.
- **Banco e storage**: PostgreSQL para metadados transacionais; object storage (S3/minio/Azure Blob) para arquivos binários; integração com TRIM via API/SDK encapsulada em adaptador.
- **Mensageria/filas**: fila (RabbitMQ/Kafka) para processamento assíncrono de digitalizações, antivírus e geração de pré-visualizações.

### Stack de tecnologias sugerida (sem Java)
- **Back-end**: Python 3.12+ com FastAPI, OAuth2/OIDC (Authlib), SQLAlchemy + Alembic para migrações, Pydantic para modelos, e uvicorn/gunicorn para runtime. Alternativa: Go 1.22 com Gin/Fiber e GORM para quem preferir tipagem estática.
- **Front-end**: SPA em React 18 + TypeScript com Next.js para SSR/SSG e roteamento protegido; ou Angular 17 caso a equipe prefira ecossistema opinionado.
- **Banco e storage**: PostgreSQL 15+, MinIO/S3; Redis para cache e sessões distribuídas.
- **Testes**: Pytest + coverage e Testcontainers; Playwright para e2e; Pact para contratos. Para Go, usar testing/testify, go test com cobertura, e Testcontainers-Go.
- **CI/CD**: GitHub Actions/GitLab CI com pipelines para build, testes, análise estática (SonarQube, Bandit/gosec), SAST/Dependency check, qualidade de código, build de imagens Docker e deploy em Kubernetes/Openshift.
- **Observabilidade**: OpenTelemetry (logs, métricas, tracing) exportando para Prometheus/Grafana e Jaeger; logs estruturados (JSON) com correalção de trace; health checks nativos de FastAPI/Go + endpoints customizados.

#### Opções de banco para documentos além do PostgreSQL
- **Metadados estruturados e consultas relacionais** continuam preferindo PostgreSQL (ou Oracle durante a transição), mas para grandes volumes de documentos/semi-estruturados é recomendável avaliar um **repositório de documentos** dedicado.
- **MongoDB**: boa opção para armazenar blobs pequenos/médios e metadados flexíveis de documentos digitalizados, com replicação/sharding nativos e consultas ad-hoc; pode coexistir com PostgreSQL para casos de uso que exigem esquema flexível (versões de formulário, anotações) ou histórico de alterações de metadados.
- **Object storage (S3/MinIO) + busca**: arquivos binários devem permanecer em object storage com versionamento; para busca full-text em campos e conteúdo, usar **OpenSearch/Elasticsearch** alimentado por pipelines de indexação a partir do storage ou do banco relacional.
- **Governança e consistência**: manter o catálogo/índices primários de documentos no banco relacional para garantir integridade referencial; MongoDB ou índices de busca ficam como repositórios complementares acessados via adaptadores específicos no domínio.

#### Por que PostgreSQL e como conviver com o banco legado
- **Motivação para PostgreSQL**: é um SGBD relacional moderno, com recursos avançados (JSONB, particionamento nativo, índices variados, janela analítica), licenciamento aberto e forte ecossistema de ferramentas. Ele reduz custos e viabiliza evolução do modelo de dados sem depender de features proprietárias do Oracle, além de integrar-se bem com ORMs e Testcontainers.
- **Compatibilidade com o banco legado**: durante a transição, o sistema deve **manter acesso pleno ao Oracle** (ou outro SGBD legado) através de adaptadores dedicados e pools separados. Nenhuma funcionalidade é descontinuada antes de existir paridade de comportamento comprovada via testes de regressão.
- **Quantas strings de conexão?**: use **duas conexões isoladas** no back-end (ex.: `datasource.oracle` e `datasource.novo`) com pools independentes. Operações que ainda dependem do schema legado usam o adaptador Oracle; novos módulos gravam no PostgreSQL. Isso evita bloqueios e facilita rollbacks graduais.
- **Migração/convivência de dados**: implementar replicação ou jobs de sincronização (CDC ou ETL incremental) para copiar metadados chave do Oracle para PostgreSQL, mantendo checksums e reconciliação automatizada. Uma camada anti-corruption no domínio mapeia diferenças de tipos/códigos e garante que a API permaneça consistente enquanto há dual-write/read.

### Stack alternativa com compatibilidade total ao banco legado
- **Back-end**: Node.js 20+ com NestJS (TypeScript) para estrutura modular e suporte a injeção de dependências; opção de Python FastAPI mantendo adaptadores Oracle.
- **Acesso ao banco legado**: drivers oficiais Oracle (node-oracledb/odpi-c) ou SQLAlchemy + cx_Oracle, encapsulados em repositórios que expõem consultas e procedures existentes sem alterar o schema; uso de connection pool (SessionPool) e mapeamentos read-only para operações que devam preservar o comportamento atual.
- **Camada de compatibilidade**: módulo de “compat Oracle” contendo DTOs compatíveis com as tabelas legadas, scripts de migração incremental e pipelines de verificação de integridade para validar leitura/gravação no banco legado antes de mover dados para PostgreSQL.
- **Front-end**: React 18 + Next.js (ou Angular) compartilhando componentes e contratos de API com a stack principal para minimizar divergência.
- **Observabilidade e segurança**: mesma abordagem da stack principal (OpenTelemetry, logs JSON, OAuth2/OIDC), incluindo auditoria específica para chamadas Oracle e métricas de pool de conexão para identificar regressões de performance.

#### Qual stack priorizar para compatibilidade com o banco legado
- **Recomendação**: priorizar a variante **NestJS + node-oracledb** quando a compatibilidade com o Oracle legado for o fator decisivo. O driver é mantido pela Oracle, oferece pooling avançado, fetch em lote e suporte pleno a tipos/PL-SQL, reduzindo risco de regressões em consultas e procedures existentes.
- **Alternativa Python**: use **FastAPI + cx_Oracle/ODPI-C** se a equipe for majoritariamente Python e já possuir pipelines nessa linguagem. A compatibilidade é sólida, mas algumas otimizações (ex.: DRCP, QoS) são mais maduras no driver Node; avalie benchmarks e testes de carga antes de optar.
- **Mitigação de riscos**: em qualquer escolha, isolar o acesso ao Oracle em adaptadores com suíte de testes de regressão sobre o schema atual, habilitar monitoramento de pool (wait time, sessions ativas) e validar comportamento de commit/rollback em cenários de concorrência.

### Padrões e boas práticas
- **DDD + Ports & Adapters** para isolar domínio de infraestrutura.
- **Autenticação federada** com OAuth2/OIDC e RBAC fino para operações de documento/anexo.
- **DTOs e validação** na borda (Bean Validation) e tratamento centralizado de erros (RFC 7807 problem details).
- **Upload seguro** com antivírus/clamav, verificação de tipo MIME e limites de tamanho; storage versionado e criptografado.
- **API idempotente** para inclusão de documentos e reprocessamento de anexos; uso de ETags/If-Match para concorrência otimista.
- **Infra como código** (Terraform/Helm), segredos em vault (HashiCorp Vault/KMS), rotacionamento de chaves.

### Requisitos não funcionais
- **Segurança**: criptografia em trânsito (TLS), em repouso (storage/banco), rotinas de hardening, auditoria de ações, proteção CSRF/XSS/SQLi, políticas de CORS e rate limiting.
- **Escalabilidade**: horizontal via contêineres/Kubernetes, cache para buscas frequentes, filas para workloads intensivos.
- **Disponibilidade/Resiliência**: readiness/liveness probes, circuit breakers/retries (Resilience4j), backups e restore testados, deploy blue/green ou canary.
- **Observabilidade/Operação**: métricas, logs estruturados, tracing, dashboards e alertas; feature flags para lançamentos graduais.

## 5. Por que a nova arquitetura é melhor
- **Isolamento e testabilidade**: camadas claras e IoC reduzem acoplamento direto observado no servlet/DAO, facilitando testes unitários e evolução contínua.【F:src/main/java/controller/ServletControlador.java†L20-L130】
- **Confiabilidade e segurança**: gestão de conexões via pool, migrações versionadas e segredos externos substituem JDBC manual com strings e credenciais embutidas.【F:src/main/java/model/DAOTrim.java†L34-L145】 Autenticação/autorizações modernas reduzem riscos de sessão frágil.
- **Experiência do usuário**: SPA responsiva com componentes modernos melhora fluxo de upload/pesquisa em relação às JSPs legadas e dependências antigas.
- **Operabilidade**: observabilidade, CI/CD e testes automatizados fornecem visibilidade e governança ausentes no legado, permitindo releases frequentes e seguros.
- **Evolução tecnológica controlada**: adotar Python/Go no back-end com React/Angular no front traz ecossistemas modernos e ampla comunidade, mantendo curva de aprendizado gerenciável e sem herdar restrições do legado em Java.

## 6. Riscos de migração e mitigação
- **Dependência do TRIM e formatos legados**: risco de incompatibilidades ao encapsular integrações; mitigar com adaptadores paralelos, testes de contrato e fase de dual run.
- **Massa de dados e arquivos**: migração de metadados e binários pode ser longa; usar migração incremental por lotes, checksums e validação de consistência.
- **Interrupção para usuários**: novas UX e APIs podem causar curva de aprendizado; planejar piloto controlado, feature flags e treinamento.
- **Integrações de scanner/clientes**: substituir executáveis locais por serviço de captura web/Desktop apoiado por fila; manter fallback até validar.
- **Mudança cultural/processual**: adoção de CI/CD, testes e monitoramento exige capacitação; conduzir coaching e definir guardrails de engenharia.

## 7. Resumo não técnico
A proposta é substituir o sistema atual por uma plataforma moderna de gestão de documentos, com uma API segura e um portal web responsivo. Os arquivos serão armazenados de forma mais confiável e rastreável, com monitoramento, auditoria e backups automatizados. O resultado esperado é maior segurança, melhor desempenho e uma experiência de uso mais simples para pesquisa e inclusão de documentos.

## 8. Visão técnica detalhada
Adotaremos uma arquitetura em camadas inspirada em hexagonal: controladores REST recebem as requisições e acionam casos de uso no núcleo de domínio. Adaptadores de saída tratam persistência em PostgreSQL (via SQLAlchemy ou ORM Go), armazenamento de arquivos em S3/MinIO e integração com TRIM através de um gateway dedicado. Serviços assíncronos em filas cuidam de digitalização, antivírus e pré-visualização. O front-end React/Next.js conversa com a API usando tokens OIDC e RBAC. A infraestrutura roda em contêineres com pipelines CI/CD automatizados, observabilidade completa (logs estruturados, métricas, tracing) e padrões de segurança (TLS, gestão de segredos, hardening) para suportar evolução contínua e alta disponibilidade.
