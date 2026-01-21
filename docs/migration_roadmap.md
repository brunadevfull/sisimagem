# Roadmap de Migração do Sistema Legado

## 1. Diagnóstico e Levantamento de Requisitos
- **Objetivos**
  - Mapear processos de negócio, regras e integrações críticas do legado.
  - Identificar dependências externas (APIs, bancos, filas, autenticação) e SLAs atuais.
  - Catalogar dados (domínios, volumetria, qualidade, retenção, privacidade) e obrigações legais.
  - Levantar dores usuais (performance, disponibilidade, segurança, manutenibilidade) para priorizar melhorias.
- **Riscos principais**
  - Lacunas de conhecimento sobre regras implícitas e exceções não documentadas.
  - Subestimação de dependências técnicas ou de licenças.
  - Datasets com qualidade baixa ou esquemas divergentes.
- **Mitigações**
  - Entrevistas com usuários-chave e operação; revisão de logs e tickets para mapear casos extremos.
  - Inventário de integrações com owners e SLAs; validar contratos e matrizes de responsabilidade.
  - Data profiling em amostras representativas; definir padrões de qualidade e planos de saneamento.

## 2. Desenho da Nova Arquitetura
- **Objetivos**
  - Definir visão alvo (domínios, bounded contexts, padrões de comunicação síncrona/assíncrona).
  - Selecionar tecnologias alinhadas a requisitos não funcionais (escalabilidade, observabilidade, segurança).
  - Estabelecer padrões transversais: versionamento de APIs, autenticação/autorização, tracing, métricas, logs estruturados.
- **Riscos principais**
  - Escolhas tecnológicas sem validação prática ou desalinhadas com a equipe.
  - Falta de governança sobre contratos e compatibilidade entre módulos.
- **Mitigações**
  - Provas de conceito curtas para tecnologias-chave; decisão com critérios claros.
  - Catálogo de padrões e playbooks; desenho de APIs versionadas e compatibilidade backward quando possível.

## 3. Desenvolvimento por Módulos (Valor x Risco)
- **Objetivos**
  - Priorizar módulos com alto valor de negócio e baixo acoplamento para entregar ganhos rápidos.
  - Implementar pipelines CI/CD, testes automatizados e feature toggles desde o primeiro módulo.
- **Riscos principais**
  - Escolha de módulos altamente acoplados atrasando entregas.
  - Ausência de automação de testes e deploy aumenta risco de regressões.
- **Mitigações**
  - Matriz Valor x Complexidade para priorização; iniciar por módulos periféricos ou front-door com pouco legado crítico.
  - Definir Definition of Done incluindo cobertura de testes, observabilidade e runbooks operacionais.

## 4. Estratégia de Migração de Dados
- **Objetivos**
  - Planejar mapeamento de esquemas, transformação e saneamento de dados.
  - Definir janelas de carga inicial, incremental (CDC/batch) e validações de consistência.
- **Riscos principais**
  - Inconsistência entre bases durante convivência.
  - Migração exceder janelas de manutenção ou impactar performance.
- **Mitigações**
  - Scripts idempotentes com checksums/contagens; dupla contagem (origem x destino) e validação de regras de negócio.
  - Testes de carga e cronogramas piloto; otimizar índices e paralelização controlada.

## 5. Estratégia de Convivência (Strangler/Parallel Run)
- **Objetivos**
  - Permitir transição gradual, roteando partes do tráfego para o novo sistema.
  - Garantir consistência entre sistemas (dual write/CDC) enquanto módulos são substituídos.
- **Riscos principais**
  - Divergência de dados ou efeitos colaterais duplicados.
  - Complexidade operacional para manter dois sistemas.
- **Mitigações**
  - Gateways com roteamento por feature flag/caminho; circuit breakers e monitoramento comparativo (shadow reads/writes controladas).
  - Janela de convivência curta e escopo bem definido; automação de rollback para o legado.

## 6. Testes e Homologação
- **Objetivos**
  - Garantir qualidade funcional e não funcional (carga, resiliência, segurança).
  - Validar migração de dados em ambiente espelho com volume realista.
- **Riscos principais**
  - Cobertura insuficiente de cenários reais e dados de borda.
  - Ambiente de homologação não representativo.
- **Mitigações**
  - Suites de testes E2E, contratos, performance e caos; dados sintéticos próximos da realidade; testes de migração dry-run.
  - Observabilidade habilitada nos ambientes de teste; KPIs de qualidade definidos e monitorados.

## 7. Corte Final para o Novo Sistema
- **Objetivos**
  - Planejar janela de virada, critérios de go/no-go e rollback.
  - Monitorar KPIs críticos na estreia (latência, erros, throughput, integridade de dados).
- **Riscos principais**
  - Tempo de indisponibilidade maior que o previsto; falha de rollback.
  - Surpresas em produção por diferenças de carga real.
- **Mitigações**
  - Playbook de cutover com passos e donos; freeze de mudanças próximas à virada; comunicações claras a usuários.
  - Script de rollback validado em ambiente de staging; war room com responsáveis de cada domínio; alertas e dashboards preparados.

---

## Checklist de Acompanhamento
- [ ] Diagnóstico concluído (processos, integrações, dados, dores)
- [ ] Arquitetura alvo validada (padrões, tecnologias, governança)
- [ ] Backlog priorizado por valor/risco e CI/CD implantado
- [ ] Estratégia de dados definida (mapas, cargas, validação)
- [ ] Plano de convivência aprovado (roteamento, sincronização, rollback)
- [ ] Testes e homologação executados (funcional, contrato, carga, caos)
- [ ] Cutover executado com monitoramento e rollback pronto
