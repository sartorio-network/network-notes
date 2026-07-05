# OPNsense — VPN site-to-site como rota de backup em cenário de link duplo

## Contexto

Ambiente de transmissão com dois sites interligados:
- **Site A (transmissor)**: recebe áudio de origem via link dedicado (fibra/rádio
  ponto-a-ponto), com sistema de distribuição de áudio via IP (ex: streaming SRT).
- **Site B (origem/matriz)**: envia o áudio de origem, além de tráfego de
  programação/administrativo, através do mesmo link dedicado.

O link dedicado (L2, extensão de LAN) é o caminho principal e não tem
substituto direto — exige que os dois lados estejam na mesma sub-rede
para o sincronismo de determinados equipamentos de broadcast. Não há
segunda via física para essa camada.

Havia, no entanto, tráfego adicional (programação, acesso a sistemas
internos) que **poderia** ser roteado via internet em caso de falha do
link principal, desde que existisse um caminho alternativo até a
sub-rede remota.

## Problema

Sem backup, qualquer falha no link dedicado causava:
- Perda total de conectividade entre os sites
- Impossibilidade de a sub-rede remota ser alcançada por qualquer
  tráfego IP, mesmo aquele que não dependia da camada L2 crítica
- Dependência de intervenção manual (acesso remoto via ferramenta de
  desktop remoto) para religar áudio de contingência na mesa/processador

## Solução

**1. WAN redundante no OPNsense do site A**
- WAN1 = perna do link dedicado (Tier 1 no Gateway Group)
- WAN2 = link de internet independente, ex: satélite/backup celular (Tier 2)
- Monitor IP da WAN1 apontando para um host do lado remoto que só
  responde se o link dedicado estiver de pé
- Monitor IP da WAN2 apontando para IP público estável (1.1.1.1, 8.8.8.8)

**2. VPN site-to-site (WireGuard) como rota de contingência**
- Túnel WireGuard entre o OPNsense do site A e o firewall do site B
- Motivo da escolha: reconexão rápida e melhor comportamento atrás de
  CGNAT (comum em links de satélite), comparado a IPsec
- Rota estática para a sub-rede remota apontando pelo túnel, com
  métrica **pior** que a rota via link dedicado — ou seja, só é usada
  quando o link principal cai
- O gateway do túnel segue o Gateway Group, então ele sobe automaticamente
  pela WAN que estiver ativa (principal ou backup)

**3. Separação clara entre "o que tem redundância" e "o que não tem"**
- Tráfego IP roteável (programação, sistemas administrativos): passa a
  ter caminho alternativo via túnel + WAN backup
- Tráfego dependente de camada L2 / sincronismo direto: **não** entra
  nesse esquema — VPN sobre um link de internet residual tende a gerar
  mais instabilidade do que resolver nesse tipo de tráfego. Nesses casos,
  a recomendação foi monitorar ativamente o link principal (alerta via
  Zabbix/Telegram) em vez de tentar simular redundância onde ela
  fisicamente não existe.

**4. Automação do chaveamento de áudio de contingência**
- O ponto manual identificado era o chaveamento de fonte de áudio via
  acesso remoto a uma mesa/processador
- Substituído (quando o hardware suporta) por detecção de silêncio
  (silence sense) configurada para trocar automaticamente da entrada
  principal para a entrada de contingência, eliminando a dependência de
  intervenção humana em tempo real durante uma falha

## Lição aprendida

Em ambientes de missão crítica, VPN sobre WAN de contingência resolve
bem tráfego IP genérico, mas não deve ser tratada como substituta de
enlaces L2 dedicados que exigem mesma sub-rede. O ganho real está em
mapear com precisão **qual tráfego pode** tolerar esse caminho alternativo
e qual não pode — e automatizar o failover até o último elo manual da
cadeia (nesse caso, o chaveamento de áudio), não só a camada de rede.
