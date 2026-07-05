# OPNsense — Failover de WAN forçando tráfego LAN indevidamente

## Contexto

Ambiente com firewall OPNsense atuando como gateway de borda, com duas saídas WAN:
- WAN primária (fibra/link principal)
- WAN secundária (Starlink/satélite, usada como failover)

Rede local segmentada, com um segmento de servidores/serviços internos
(ex: `10.0.0.0/22`) que não deveria trafegar pela WAN de failover em
nenhuma circunstância.

## Problema

Uma regra de firewall genérica, criada para garantir failover automático
("SAIDA-COM-FAILOVER"), estava definida de forma ampla demais:

- Origem: LAN (qualquer)
- Destino: any
- Gateway: grupo de failover (WAN_PRIMARY / WAN_BACKUP)

Como essa regra ficava **acima** de qualquer regra mais específica na
ordem de avaliação, todo o tráfego da LAN — incluindo tráfego destinado
a servidores internos — estava sendo roteado através do grupo de
gateway de failover, mesmo quando o destino era local.

Resultado: tráfego interno (ex: acesso a servidores de aplicação, DNS
interno, backups) saindo pela WAN secundária de forma desnecessária,
gerando lentidão e uso indevido de um link de contingência (link de
satélite, com throughput e latência inferiores).

## Diagnóstico

1. Verificação de "States" ativos no OPNsense (`Firewall > Diagnostics > States`)
   mostrou sessões para IPs internos sendo associadas ao gateway de failover.
2. Confirmado que a regra de failover não possuía exceção para o
   range de rede interna.
3. Ordem das regras confirmada em `Firewall > Rules > LAN` — regra
   genérica posicionada antes de qualquer regra específica de destino interno.

## Solução

Criar uma regra **específica**, posicionada **acima** da regra genérica
de failover, para o tráfego LAN → rede interna, **sem override de
gateway** (ou seja, usando o gateway padrão do sistema, não o grupo de
failover):

Regra 1 (topo):
Interface: LAN
Origem: LAN net
Destino: 10.0.0.0/22  (rede interna de servidores)
Gateway: default (sem override)
Ação: Allow
Regra 2 (abaixo, já existente):
Interface: LAN
Origem: LAN net
Destino: any
Gateway: FAILOVER_GROUP (WAN_PRIMARY / WAN_BACKUP)
Ação: Allow

omo o OPNsense (baseado em pf) avalia regras de cima para baixo e
aplica a **primeira regra correspondente**, o tráfego para a rede
interna passa a ser resolvido localmente, sem gateway de failover, e
apenas o tráfego realmente destinado à internet segue pela lógica de
failover.

## Lição aprendida

Regras de failover "catch-all" devem sempre ser posicionadas **depois**
de regras específicas para destinos internos. Em ambientes de missão
crítica (broadcast, transmissão ao vivo), esse tipo de erro de ordenação
pode causar lentidão perceptível em sistemas internos sem que o link
principal esteja de fato indisponível — o que dificulta o diagnóstico
inicial, já que "a internet está funcionando" mas serviços internos
ficam degradados.
