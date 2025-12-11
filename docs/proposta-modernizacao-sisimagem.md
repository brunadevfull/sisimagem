# Proposta de Modernização do SisImagem

## 1. Contexto
### 1.1 Descrição breve do sistema atual
O SisImagem é uma aplicação web monolítica que concentra autenticação, pesquisa e inclusão de documentos, armazenando metadados no Oracle/TRIM e arquivos em diretório local do servidor de aplicação on-premises. A navegação ocorre por páginas JSP servidas por servlets e jQuery 1.4.2 para interações básicas no navegador.

### 1.2 Principais limitações do sistema legado
- **Arquitetura monolítica e acoplada**: difícil de evoluir modularmente, exige implantação completa a cada alteração e depende de servidor de aplicação compatível com Java EE 7.
- **UI defasada e pouco responsiva**: baseada em JSP/jQuery antigo, com experiência de usuário inconsistente em dispositivos modernos.
- **Integrações rígidas e on-premises**: conexão direta via JDBC ao Oracle/TRIM com configuração embutida, dependência de diretório de arquivos local e de aplicativo de scanner desktop, dificultando mobilidade e alta disponibilidade.
- **Segurança e conformidade**: autenticação própria, ausência de MFA/SSO, gestão de credenciais sensível a vazamentos e sem observabilidade centralizada.
- **Escalabilidade limitada**: falta de containerização e automação de infraestrutura, dificultando elasticidade e ambientes paralelos de teste/homologação.
- **Custos operacionais e de suporte**: troubleshooting depende de logs em arquivo, builds manuais e pouca cobertura de testes automatizados, gerando risco de regressões e maior tempo de resposta.

## 2. Proposta
### 2.1 Visão geral da nova solução
Reescrever o SisImagem como uma plataforma modular em arquitetura de microsserviços orientada a eventos, com front-end SPA responsivo e APIs REST/GraphQL padronizadas. A solução priorizará desacoplamento, segurança corporativa, observabilidade e pipelines CI/CD que permitam releases frequentes com qualidade, utilizando uma stack moderna sem dependência de Java. Em termos simples, cada parte crítica do sistema passa a ser um serviço independente, fácil de atualizar e escalável conforme a demanda.

### 2.2 Tecnologias propostas
- **Back-end**: microserviços em Node.js 20 com NestJS (TypeScript), APIs REST e GraphQL, segurança com OpenID Connect (Keycloak/ADFS/Azure AD) e mensageria com Kafka para fluxos assíncronos (ex.: indexação, notificações, ETL).
- **Front-end**: SPA em React 18 + TypeScript, design system corporativo, React Query para caching de dados, componentes responsivos e acessíveis (WCAG 2.1).
- **Banco de dados**: Postgres para dados transacionais; armazenamento de objetos on-premises compatível com S3 (ex.: MinIO) para documentos; mecanismo de busca full-text com OpenSearch/Elasticsearch para consultas rápidas.
- **Infraestrutura**: containerização com Docker e orquestração leve com Docker Compose ou Swarm em data center próprio; provisionamento automatizado com Ansible/Terraform para VMs; feature flags para releases seguros; observabilidade com Prometheus/Grafana/Loki e tracing com OpenTelemetry.
- **CI/CD**: pipelines em GitHub Actions/GitLab CI com build, testes unitários/integração, scans de qualidade (SonarQube), SAST/DAST, geração de artefatos e deploy automatizado para ambientes dev/homolog/produção.

### 2.3 Benefícios diretos
- **Manutenção e evolutividade**: serviços pequenos, bem separados e fáceis de testar, reduzindo risco de regressão e acelerando entregas.
- **Segurança**: autenticação centralizada com MFA/SSO, gestão protegida de segredos e trilhas de auditoria para conformidade.
- **Performance e escalabilidade**: capacidade de escalar horizontalmente os serviços on-premises conforme a demanda, uso de cache e filas para manter tempo de resposta baixo mesmo com mais usuários.
- **Experiência do usuário**: interface moderna, responsiva e com notificações em tempo real; suporte pleno a dispositivos móveis.
- **Integração futura**: APIs padronizadas e eventos facilitam conexão com outros sistemas corporativos, preservando rastreabilidade.
- **Resiliência operacional**: implantações graduais (blue/green, canary), monitoramento em tempo real e mecanismos de tolerância a falhas.

## 3. Comparação Legado x Novo
- **Arquitetura**: monolito Java EE em servidor de aplicação **→** microsserviços em Node.js/NestJS orquestrados on-premises com Docker Compose/Swarm.
- **Front-end**: JSP + jQuery 1.4.2 **→** React/TypeScript com design system e build moderno.
- **Banco/arquivos**: Oracle/TRIM + diretório local **→** Postgres + storage compatível com S3 on-premises + busca OpenSearch.
- **Autenticação**: login próprio e sessões em servidor **→** OpenID Connect, tokens JWT e MFA.
- **Implantação**: builds manuais em WAR **→** pipelines CI/CD com imagens Docker e deploy automatizado para VMs on-premises.
- **Observabilidade**: logs em arquivo local **→** métricas, logs centralizados e tracing distribuído.
- **Escalabilidade**: instância única ou cluster manual **→** escalonamento horizontal controlado em contêineres on-premises e alta disponibilidade.
- **Experiência**: formulários estáticos **→** SPA responsiva com UX consistente e suporte mobile.

## 4. Plano de Migração
1. **Levantamento e arquitetura**: mapear domínios, fluxos críticos (login, busca, inclusão, anexos) e dependências externas; definir módulos de negócio e contratos de API claros.
2. **Prova de conceito**: implementar núcleo de autenticação via OpenID Connect e um serviço de documentos mínimo (CRUD + upload em storage compatível com S3 on-premises) com front-end React básico.
3. **Faseamento por módulos**: entregar serviços em ondas (autenticação/usuários, documentos/metadados, anexos/arquivos, pesquisa full-text, auditoria) com gateways de compatibilidade REST.
4. **Dados e integrações**: planejar ETL de metadados do Oracle/TRIM para Postgres e migração de arquivos para storage compatível com S3 on-premises; criar conectores para coexistência temporária com o TRIM durante a transição.
5. **Testes e qualidade**: testes unitários/integração, contratos de API, testes end-to-end e desempenho; habilitar SAST/DAST na pipeline.
6. **Transição gradual**: adotar estratégia strangler, roteando partes do tráfego pelo API Gateway para novos serviços; manter operação do legado até completar migração de cada fluxo.
7. **Go-live e suporte**: executar blue/green, monitorar KPIs (latência, erros, throughput, adesão MFA) e estabelecer runbooks e acordos de nível de serviço.

### Cuidados para não interromper a operação
- Manter rotas compatíveis via API Gateway/Reverse Proxy enquanto front-end é trocado gradualmente.
- Sincronizar dados entre TRIM e o novo storage durante a coexistência para evitar perda de documentos.
- Implementar feature flags e toggles de roteamento para rollback rápido em caso de falhas.
- Planejar janelas de migração de dados com replicação incremental e validação pós-carga.

## 5. Conclusão
A modernização proposta transforma o SisImagem em uma plataforma segura, escalável e preparada para integração, reduzindo risco operacional e acelerando entregas de novas funcionalidades. O investimento habilita ganhos estratégicos — produtividade das equipes, experiência superior para usuários e governança de informação aderente às exigências corporativas e regulatórias — garantindo longevidade e competitividade ao sistema.
