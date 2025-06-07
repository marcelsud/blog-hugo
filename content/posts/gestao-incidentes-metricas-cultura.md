---
title: "Gestão de Incidentes: Métricas que Importam e a Cultura por Trás Delas"
date: 2025-01-07
draft: false
categories: ["SRE", "DevOps", "Gestão de Incidentes"]
tags: ["incidentes", "metricas", "MTTR", "MTTD", "cultura-blameless", "sre", "observabilidade"]
---

Depois de passar madrugadas inteiras apagando incêndios em produção e participar de centenas de post-mortems, aprendi uma verdade inconveniente: a maioria das empresas mede as métricas erradas e cria uma cultura que incentiva esconder problemas ao invés de resolvê-los. Vamos falar sobre o que realmente importa na gestão de incidentes.

## As Métricas que Todo Mundo Conhece (Mas Poucos Entendem)

### MTTD - Mean Time To Detect

O tempo médio para detectar um incidente. Parece simples, certo? Errado. A pegadinha está em definir quando o incidente realmente começou. Foi quando o primeiro erro apareceu nos logs? Quando o primeiro cliente reclamou? Quando o alerta disparou?

**Na prática**: Se seu MTTD é maior que 5 minutos, você provavelmente está descobrindo problemas pelo Twitter. Sim, aquele momento constrangedor quando o CEO te manda print de um tweet reclamando que o sistema está fora.

### MTTR - Mean Time To Recovery

O tempo médio para resolver um incidente. Esta é a métrica favorita dos executivos porque parece indicar eficiência. Mas cuidado: um MTTR baixo pode significar tanto excelência operacional quanto gambiarras perigosas.

**Exemplo real**: Uma equipe que conheço tinha MTTR de 15 minutos. Impressionante? Nem tanto. Eles simplesmente reiniciavam os servidores a cada problema. O "incidente" sumia, mas a causa raiz continuava lá, como uma bomba-relógio.

### MTTF - Mean Time To Failure

O tempo médio entre falhas. Esta métrica é cruel porque mostra a real confiabilidade do seu sistema. Um MTTF alto significa que você dorme tranquilo. Um MTTF baixo significa que você conhece todos os entregadores de pizza da região.

### MTBF - Mean Time Between Failures

Similar ao MTTF, mas inclui o tempo de reparo. É a métrica que mostra o ciclo completo: funcionando → quebrado → consertado → funcionando de novo.

## As Métricas que Realmente Importam (E Ninguém Fala)

### Taxa de Reincidência

Quantos incidentes são recorrentes? Se o mesmo problema volta toda semana, seu MTTR baixo não vale nada. É como tomar remédio para dor de cabeça causada por um tumor - você está tratando o sintoma, não a doença.

```python
# Pseudo-código para tracking de reincidência
def calcular_taxa_reincidencia(incidentes):
    incidentes_unicos = set()
    incidentes_recorrentes = 0
    
    for incidente in incidentes:
        fingerprint = gerar_fingerprint(incidente)
        if fingerprint in incidentes_unicos:
            incidentes_recorrentes += 1
        else:
            incidentes_unicos.add(fingerprint)
    
    return incidentes_recorrentes / len(incidentes) * 100
```

### Tempo para Post-Mortem Completo

Não é o tempo para escrever o documento, mas para implementar todas as ações. Um post-mortem sem follow-up é só storytelling corporativo.

### Customer Impact Time (CIT)

O tempo total que clientes foram impactados, não apenas o tempo do incidente. Se você levou 1 hora para resolver mas os clientes sentiram efeitos por 3 horas (cache, filas, reprocessamento), seu CIT é 3 horas.

## A Resistência Cultural: Por Que Ninguém Quer Abrir Incidentes

### O Medo do Julgamento

"Quem abriu o incidente provavelmente causou o problema" - essa mentalidade tóxica ainda existe em muitas empresas. O resultado? Problemas são varridos para debaixo do tapete até virarem crises.

**História real**: Em uma empresa, descobri que os devs criaram um canal secreto no Slack chamado #problema-que-nao-e-incidente onde discutiam issues em produção sem abrir tickets oficiais. Por quê? O gerente cobrava explicações de quem abria incidentes.

### A Síndrome do Herói

Algumas culturas valorizam mais quem apaga incêndios do que quem previne incêndios. É o bombeiro piromaníaco corporativo: cria o problema para depois ser o herói que resolve.

### Métricas Punitivas

Quando o bônus está atrelado a "zero incidentes", adivinha o que acontece? Zero incidentes reportados, não zero problemas.

## Construindo uma Cultura Blameless de Verdade

### 1. Celebrate a Transparência

Na Netflix, há um princípio: "Sunshine is the best disinfectant". Problemas expostos são problemas resolvidos. Implementei algo similar:

- **Incident Hero Award**: Reconhecimento mensal para quem detectou problemas críticos cedo
- **Near Miss Reports**: Valorizar quem reporta "quase incidentes"
- **Public Post-Mortems**: Compartilhar aprendizados com toda empresa, não só com a engenharia

### 2. Mude a Linguagem

Palavras importam. Pare de perguntar "quem causou isso?" e comece com "o que podemos aprender?". 

**Antes**: "Por que você deployou código bugado?"
**Depois**: "Que gaps no nosso processo permitiram que código com defeito chegasse em produção?"

### 3. Implemente Follow-ups Estruturados

Post-mortem sem ação é só desabafo coletivo. Criei um processo simples mas eficaz:

```yaml
post_mortem_lifecycle:
  - incident_closed:
      deadline: "48 horas para post-mortem draft"
  - review_meeting:
      participants: ["time envolvido", "SRE", "product owner"]
      output: "action items com owners e deadlines"
  - weekly_check:
      review: "status de todos action items"
      escalation: "items atrasados vão para leadership"
  - monthly_retrospective:
      analyze: "tendências e patterns"
      adjust: "processos e ferramentas"
```

### 4. Torne Incidentes Boring

Quanto mais rotineiro e processual for lidar com incidentes, menos drama haverá. Isso significa:

- **Runbooks automatizados**: Se é repetitivo, deve ser automatizado
- **Roles claros**: Incident Commander, Comunicador, Investigadores
- **Templates padronizados**: Nada de reinventar a roda a cada incidente

## Exemplos Práticos de Implementação

### Caso 1: E-commerce com Picos Sazonais

**Problema**: Black Friday sempre derrubava o sistema
**Métricas tradicionais**: MTTR de 2 horas (péssimo)
**Nossa abordagem**:
- Criamos "Game Days" mensais simulando Black Friday
- Métrica nova: "Capacidade de absorção de pico" 
- Resultado: Última Black Friday com zero downtime

### Caso 2: Fintech com Zero Tolerância a Erros

**Problema**: Medo de reportar incidentes por regulamentação
**Métricas tradicionais**: MTTD de 45 minutos (clientes ligando)
**Nossa abordagem**:
- Diferenciamos "incidentes" de "eventos de observabilidade"
- Criamos níveis: SEV1 (crítico) até SEV5 (melhoria)
- Incentivamos reporte de SEV4/5 sem burocracia
- Resultado: MTTD caiu para 3 minutos

### Caso 3: Startup em Hypergrowth

**Problema**: Cada semana um serviço novo, cada semana um incidente novo
**Métricas tradicionais**: MTTF de 3 dias (caótico)
**Nossa abordagem**:
- "Chaos Engineering Fridays": quebrar coisas controladamente
- Métrica nova: "Tempo para primeira falha em produção"
- Se algo novo não falha em 30 dias, quebramos de propósito
- Resultado: MTTF subiu para 3 semanas

## A Verdade Sobre Follow-ups

A maioria dos post-mortems morre na praia dos action items. Por quê? Porque tratamos follow-up como trabalho extra, não como parte do trabalho.

**Framework que funciona**:

1. **20% do sprint para débito técnico**: Isso inclui action items de incidentes
2. **Owner != Executor**: Quem é dono do action item não precisa implementar, mas precisa garantir que aconteça
3. **Sunset automático**: Action items não implementados em 90 dias são revisados - ainda são relevantes?

## Conclusão: Métricas São Meios, Não Fins

No final do dia, métricas são ferramentas, não objetivos. Uma cultura saudável de gestão de incidentes se baseia em:

- **Aprendizado contínuo** ao invés de punição
- **Transparência radical** ao invés de política corporativa
- **Prevenção proativa** ao invés de heroísmo reativo
- **Melhoria sistêmica** ao invés de band-aids

Se sua empresa ainda trata incidentes como fracassos ao invés de oportunidades de aprendizado, está na hora de mudar. Comece pequeno: celebre o próximo pessoa que detectar um problema. Agradeça publicamente quem abrir um incidente. Transforme post-mortems em sessões de aprendizado, não tribunais.

Lembre-se: em sistemas complexos, falhas são inevitáveis. O que diferencia empresas maduras das amadoras não é a ausência de incidentes, mas como elas respondem a eles.

E se você ainda mede sucesso por "dias sem incidentes", tenho más notícias: ou seu sistema é trivial, ou seus incidentes estão sendo escondidos. Em ambos os casos, você tem um problema maior do que imagina.

---

*No fim das contas, a melhor métrica de todas é simples: sua equipe tem medo ou empolgação quando o alerta de incidente toca? Se é medo, você está medindo as coisas erradas.*