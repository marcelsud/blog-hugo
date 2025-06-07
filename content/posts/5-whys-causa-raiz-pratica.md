---
title: "5 Whys na Prática: Encontrando a Causa Raiz Sem Perder a Sanidade"
date: 2025-01-07
draft: false
categories: ["SRE", "DevOps", "Gestão de Incidentes"]
tags: ["5-whys", "root-cause-analysis", "RCA", "post-mortem", "incidentes", "troubleshooting"]
---

A técnica dos 5 Whys é como bisturi na mão de cirurgião: poderosa quando bem usada, desastrosa quando não. Depois de facilitar dezenas de sessões de RCA (Root Cause Analysis), vi de tudo: desde análises brilhantes que preveniram desastres futuros até sessões que viraram caça às bruxas ou, pior ainda, teatro corporativo. Vamos ao que realmente funciona.

## O Problema com os 5 Whys Tradicionais

A técnica, criada pela Toyota, parece simples: pergunte "por quê?" cinco vezes e chegue à causa raiz. Na prática, é como dizer que para ser rico basta comprar na baixa e vender na alta - tecnicamente correto, praticamente inútil sem contexto.

**Exemplo clássico (e inútil)**:
```
Problema: Site fora do ar
Por quê? Servidor web parou
Por quê? Ficou sem memória  
Por quê? Memory leak no código
Por quê? Desenvolvedor não testou direito
Por quê? Estava com pressa
```

Parabéns, você acabou de culpar o João e não resolveu nada. 

## 5 Whys que Funcionam: Casos Reais

### Caso 1: O Mistério dos Uploads Fantasmas

**Contexto**: Sistema de upload de documentos falhando aleatoriamente às terças-feiras.

**Investigação**:
```yaml
Problema: Uploads falham às terças, ~14h
↓ Por quê?
API retorna timeout após 30 segundos
↓ Por quê?
Processo de vírus scan demora mais que o normal
↓ Por quê? 
Base de assinaturas de vírus muito grande às terças
↓ Por quê?
Update semanal de assinaturas acontece terça 13h30
↓ Por quê?
Cron foi configurado em UTC, não em horário local
```

**Causa Raiz Real**: Problema de timezone, não de performance.

**Ações**:
1. Ajustar cron para horário de menor uso (madrugada)
2. Implementar cache de assinaturas já verificadas
3. Alertas para degradação de performance em uploads

### Caso 2: Database Connection Pool do Inferno

**Contexto**: Aplicação começou a ter spikes de latência após deploy "sem mudanças".

```yaml
Problema: Latência de 5s em requests que eram 200ms
↓ Por quê?
Connections com DB demoram para ser estabelecidas
↓ Por quê?
Pool de conexões está sempre vazio
↓ Por quê?
Health check fecha conexões após teste
↓ Por quê?
Nova lib de health check não reutiliza conexões do pool
↓ Por quê?
Dev copiou exemplo da doc sem adaptar ao nosso contexto
```

**Plot Twist**: O "deploy sem mudanças" atualizou dependências transitivas.

**Ações**:
1. Lock de versões em dependências críticas
2. Health check com conexão dedicada fora do pool
3. Métricas de utilização do connection pool

### Caso 3: O Paradoxo do Cache Perfeito

**Contexto**: Sistema mais lento DEPOIS de implementar cache.

```yaml
Problema: API 3x mais lenta com cache Redis
↓ Por quê?
Cada request faz múltiplas chamadas ao Redis
↓ Por quê?
Cache granular demais (1 key por campo)
↓ Por quê?
Implementação seguiu pattern de ORM cegamente
↓ Por quê?
Arquiteto definiu "cache em todas as camadas"
↓ Por quê?
Trauma de incidente anterior sem cache
```

**Lição**: Às vezes a causa raiz é overcorrection de problema passado.

## Quando Parar de Perguntar "Por Quê?"

A regra dos "5" é arbitrária. O segredo é parar quando você chega em uma das seguintes situações:

### 1. Ação Concreta Possível
Se você pode fazer algo específico para prevenir o problema, pare.

**Bom**: "Config de timeout muito baixo" → Aumentar timeout
**Ruim**: "Cultura da empresa" → ???

### 2. Limite de Controle
Quando sai da sua esfera de influência, documente mas não continue.

**Exemplo**: 
```yaml
Por quê AWS teve outage?
→ Problema no datacenter deles
→ PARE. Você não vai consertar a AWS.
```

### 3. Loop Circular
Se as respostas começam a se repetir, você foi longe demais.

```yaml
Por quê não temos testes?
→ Não temos tempo
Por quê não temos tempo?
→ Muitos bugs para corrigir
Por quê muitos bugs?
→ Não temos testes
→ 🔄 LOOP DETECTADO
```

## Como Evitar Análises Superficiais

### 1. Use Dados, Não Opiniões

**Superficial**: "O deploy causou o problema"
**Profundo**: "Commit abc123 introduziu query N+1 que aparece no slow query log"

### 2. Diversifique os "Por Quês"

Não use apenas "Por quê?". Varie:
- "O que mudou?"
- "Quando começou?"
- "Onde mais isso acontece?"
- "Como sabemos disso?"
- "Quem mais foi impactado?"

### 3. Timeline Reversa

Comece do momento da detecção e vá voltando:
```
14:32 - Alerta disparou
14:30 - Erro rate subiu para 15%
14:28 - Deploy em produção
14:15 - PR merged sem aprovação
13:45 - Tests desabilitados "temporariamente"
```

### 4. Embrace o "Não Sei"

Melhor admitir ignorância do que inventar causa:

```yaml
Por quê memory leak aconteceu?
→ Não sabemos ainda, precisamos de:
  - Memory profiler em produção
  - Heap dumps do momento do problema
  - Reproduzir em ambiente controlado
```

## Exemplos de RCA Bem Documentado

### Template que Funciona

```markdown
# RCA: Outage Sistema de Pagamentos - 2024-03-15

## Impacto
- Duração: 45 minutos (14:00 - 14:45 UTC)
- Clientes afetados: ~15.000
- Transações perdidas: 0 (todas foram enfileiradas)
- Impacto financeiro: R$ 45.000 em vendas atrasadas

## Timeline
14:00 - Deploy automático após merge
14:02 - Primeiros timeouts em logs
14:05 - Alerta de latência disparou
14:08 - Equipe iniciou investigação
14:15 - Identificado lock em tabela
14:20 - Rollback iniciado
14:30 - Sistema restaurado
14:45 - Backlog processado

## Análise 5 Whys

Problema: Todas transações travaram por 45min
↓ Por quê?
Lock exclusivo na tabela de transações
↓ Por quê?
Migration rodou ALTER TABLE em produção
↓ Por quê?
Deploy incluiu migration não revisada
↓ Por quê?
CI/CD não diferencia migrations perigosas
↓ Por quê?
Assumimos que todas migrations são safe

## Causa Raiz
Migration com ALTER TABLE bloqueante em tabela crítica foi executada automaticamente em horário de pico.

## Ações
1. [P0] Migrations com ALTER TABLE requerem aprovação manual
2. [P0] Implementar pt-online-schema-change para alterações
3. [P1] Alertas para locks longos em tabelas críticas
4. [P1] Runbook para situações de lock
5. [P2] Workshop sobre migrations safe para todo time

## Lições Aprendidas
- Nem toda migration é criada igual
- Ferramentas de schema change online são essenciais
- Automação sem safeguards é uma arma carregada
```

## Como Compartilhar Aprendizados

### 1. RCA Lightning Talks
5 minutos, 3 slides:
- O que aconteceu
- Por que aconteceu  
- O que fizemos para nunca mais acontecer

### 2. Failure Friday
Sessão mensal onde times compartilham suas "melhores" falhas. Regras:
- Sem julgamentos
- Foco no aprendizado
- Prêmio para "Falha Mais Educativa"

### 3. Runbook Wiki
Cada RCA vira um runbook:
```markdown
## Se você ver: "Lock wait timeout exceeded"

### Diagnóstico Rápido
```sql
SHOW PROCESSLIST;
SELECT * FROM information_schema.innodb_locks;
```

### Ação Imediata
1. Identificar query bloqueante
2. Se migration: KILL <process_id>
3. Se transação business: escalar para produto

### Prevenção
- Sempre use pt-online-schema-change
- Rode migrations fora do horário de pico
- Teste em staging com dados reais
```

## Armadilhas Comuns e Como Evitar

### 1. A Culpa é do Estagiário
Se seu 5 Whys sempre termina em "erro humano", você está fazendo errado.

**Errado**: "João não testou"
**Certo**: "Nosso processo permite deploy sem testes"

### 2. Paralisia por Análise
Não precisa entender física quântica para consertar um bug.

**Pragmático**: Pare quando tiver ação clara
**Acadêmico**: Continue até entender o universo

### 3. RCA Como Punição
Se RCA vira tribunal, pessoas param de reportar problemas.

**Criar**: Ambiente de aprendizado
**Evitar**: Caça às bruxas

## Ferramentas que Facilitam

### Para Coleta de Evidências
```bash
# Script que rodo assim que incidente começa
#!/bin/bash
INCIDENT_DIR="incident-$(date +%Y%m%d-%H%M%S)"
mkdir $INCIDENT_DIR
kubectl logs -n prod --tail=1000 > $INCIDENT_DIR/k8s-logs.txt
pg_dump -s > $INCIDENT_DIR/db-schema.sql
curl http://metrics/api/snapshot > $INCIDENT_DIR/metrics.json
git log --oneline -20 > $INCIDENT_DIR/recent-commits.txt
```

### Para Visualização
- **Mermaid**: Para diagramas de sequência
- **draw.io**: Para diagramas de arquitetura
- **Grafana**: Para correlacionar métricas com eventos

## Conclusão: RCA é Sobre o Futuro, Não o Passado

O objetivo do 5 Whys não é encontrar culpados ou provar que você é o Sherlock Holmes da engenharia. É sobre prevenir que o mesmo problema aconteça novamente.

Uma boa análise de causa raiz:
- Foca em sistemas, não pessoas
- Busca múltiplas causas contribuintes
- Propõe ações concretas e mensuráveis
- É compartilhada amplamente
- Vira conhecimento institucional

Se seu 5 Whys não está gerando mudanças reais, você está apenas fazendo teatro. E se está virando sessão de apontar dedos, está na hora de repensar sua cultura.

Lembre-se: todo sistema complexo falha de formas complexas. A arte está em aprender com cada falha sem perder a sanidade no processo.

---

*No final, a melhor pergunta não é "por que isso aconteceu?" mas sim "o que podemos fazer para que seja impossível acontecer de novo?". Essa mudança de perspectiva transforma post-mortems de autópsias em sessões de fortalecimento do sistema.*