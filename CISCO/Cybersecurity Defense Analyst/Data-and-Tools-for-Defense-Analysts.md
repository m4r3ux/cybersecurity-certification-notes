## 1. Contexto: Wonderland SOC

- Cenário fictício de treinamento: **Wonderland SOC**, um MSSP (Managed Security Service Provider — provedor terceirizado de serviços de segurança gerenciada).
- Um MSSP presta serviço de segurança para **várias empresas ao mesmo tempo**, monitorando diferentes ambientes.
- Estrutura em **tiers (níveis)** de analistas: alertas são revisados, resolvidos ou escalados para o próximo nível.
- Mentor do treinamento: Robin, analista Tier 2.

### Papel do analista de SOC

- Detecção e resposta contínua (cobertura 24h).
- Analisar incidentes ajuda a identificar pontos fracos na postura de segurança do cliente.
- O monitoramento contínuo fortalece a resiliência contra ameaças.

---

## 2. Teorias de Defesa (Defense Theories)

Fundamentam as arquiteturas e ferramentas escolhidas por uma organização, com base em: tolerância a risco, objetivos de negócio e requisitos regulatórios.

|Teoria|Ideia central|
|---|---|
|**Defense in Depth**|Múltiplas camadas de controle. Se uma camada é comprometida, outras continuam protegendo.|
|**Zero Trust**|(mencionada, não detalhada no material)|
|**Active Defense**|(mencionada, não detalhada no material)|
|**Resilience Theory**|(mencionada, não detalhada no material)|

> Nota: na prática, essas teorias costumam ser **combinadas** dentro de uma mesma organização.

---

## 3. Ferramentas usadas por analistas

Controles de segurança podem ser:

- **Administrativos** (ex.: políticas de uso autorizado)
- **Técnicos** (ex.: firewall)
- **Físicos** (ex.: cerca)

Dentro do SOC, o foco maior é em controles **detectivos** e **corretivos**.

### SIEM (Security Information and Event Management)

- Ferramenta central do SOC (Wonderland usa **Splunk Enterprise Security — ES**).
- Suporta: análise de eventos, integração de threat intelligence, detecção de ameaças.
- Usa **correlation searches**: buscas automatizadas que cruzam dados para gerar **notable events** (eventos notáveis) quando algo suspeito é encontrado.

**Exemplo de correlação:** Múltiplas tentativas de login falhas do mesmo usuário + logins de localizações geográficas diferentes → isoladamente podem ser normais, mas juntas geram um notable event (maior probabilidade de ser malicioso).

### Automação

- Reduz a carga de trabalho do analista automatizando tarefas investigativas padrão.
- Auxilia também na resposta a incidentes.

### Apps e Add-ons (Splunkbase)

||Apps|Add-ons|
|---|---|---|
|Uso|Visualização, análise, representação|Otimização e coleta de dados|
|Interface (GUI)|Sim|Não|

- Splunkbase (splunkbase.splunk.com): marketplace de apps/add-ons para Splunk e apps para SOAR. Comunidade pode publicar os próprios.

---

## 4. Fontes de Dados de Segurança (Data for Defense)

Uma arquitetura em camadas gera **múltiplas fontes de dados relevantes** para investigação.

### 4.1 Tráfego de Rede (Network Traffic)

- **Wire data**: dados brutos trafegando na rede.
- **Packet capture (pcap)**: cópia completa do tráfego, capturado bit a bit. Muito detalhado, porém volumoso.
    - Bibliotecas/API: `libpcap` (Unix), `WinPcap` (legado Windows), `Npcap` (atual, Windows 7+).
- Alternativas mais leves ao pcap completo:
    - **NetFlow**: protocolo da Cisco (RFC 3954), por dispositivo.
    - **VPC Flow Logs**: AWS/Google Cloud.
    - **NSG Flow Logs**: Azure Network Watcher.
    - Fornecem menos detalhe que pcap completo, mas são úteis para: alertar sobre problemas, correlacionar informações, identificar tendências.

**Exemplo — leitura de pcap (Wireshark):**

- Consultas DNS revelam sites acessados pelo usuário.
- IP do gateway/servidor DNS pode ser inferido pela sub-rede (ex.: `192.168.0.1` = gateway; `192.168.0.210` = máquina do usuário).
- Pacotes "Standard query" / "Standard query response" = processo do protocolo DNS.
- Uma consulta a um domínio de vídeo logo após acessar um site pode indicar vídeo incorporado na página.

**Exemplo — VPC Flow Log (AWS):**

```
2 123456789010 eni-1234567890abc1234 172.31.16.1 172.31.16.200 20641 22 6 20 4249 1418530010 1418530070 REJECT OK
```

- Tráfego SSH (porta destino 22, protocolo TCP/6) de `172.31.16.1` para a interface `eni-...` (IP privado `172.31.16.200`) na conta AWS `123456789010`.
- Resultado: **REJECT** (bloqueado por security group).

**DPI (Deep Packet Inspection):**

- Inspeção do payload do tráfego **em tempo real**.
- Alto custo computacional (grande volume de dados) — nem sempre viável.
- Tráfego criptografado exige **interceptação TLS**, que é cara e é, ela mesma, um risco de segurança (ver alerta da CISA: "HTTPS Interception Weakens TLS Security").

---

### 4.2 IDS / IPS (Intrusion Detection/Prevention Systems)

- Monitoram tráfego de rede em busca de atividade suspeita **baseada em assinaturas (signatures)**.
- IDS = detecta e alerta. IPS = detecta e bloqueia. Muitos sistemas fazem as duas coisas.
- **Baseados em rede**: Snort, Suricata, Cisco Firepower NGIPS, Palo Alto Networks.
- **Baseados em host**: McAfee, OSSEC, Tripwire.

**Exemplo — log do Snort (possível SYN flood / DoS):**

json

```json
{
  "timestamp": "06/17-21:53:38.555249",
  "class": "Attempted Denial of Service",
  "msg": "Possible DoS Attack Type : SYN flood",
  "priority": 2,
  "src_addr": "172.16.0.250",
  "src_port": 37396,
  "dst_addr": "192.168.1.6",
  "dst_port": 80
}
```

- Porta destino 80 → servidor de destino provavelmente hospeda serviço web.
- Ambos IPs são de faixas privadas → o ataque partiu de **dentro** da própria empresa. Se não for teste de segurança planejado, é motivo de preocupação.

---

### 4.3 Firewalls

- Protegem fronteiras entre redes protegidas e redes menos seguras.
- Logs mostram quais ativos se comunicaram (ou tentaram) dentro do ambiente.

**O que buscar em logs de firewall:**

- Tráfego/conexões permitidas fora do padrão entre zonas protegidas.
- Atividade de protocolo incomum.
- Negações excessivas vindas de uma única origem ou direcionadas a um host/rede.
- Evidência de port scanning seguida de abertura de portas.

**Exemplo — log Juniper Firewall (encerramento de sessão TCP CLIENT RST):**

```
source-address="172.16.50.200" source-port="63608"
destination-address="52.216.129.45" destination-port="443"
service-name="junos-https" application="SSL"
nat-source-address="135.84.144.253" nat-source-port="3403"
source-zone-name="Internal" destination-zone-name="Internet"
```

Pontos-chave:

- Firewalls geralmente bloqueiam tudo que não é explicitamente permitido → deve existir uma regra liberando esse tráfego Internal → Internet.
- **NAT** (`nsw-src-interface`): traduz IP interno (`172.16.50.200`) para IP público (`135.84.144.253`); porta de origem também é traduzida (`63608` → `3403`).
- Porta destino 443 = conexão HTTPS (criptografada); `service-name` indica tráfego para AWS.
- Sessão encerrada porque o cliente finalizou a comunicação (`TCP CLIENT RST`).

---

### 4.4 Endpoints

- Desktops, laptops, dispositivos móveis, wearables etc.
- Alta fidelidade de log → grande insight sobre táticas e técnicas do adversário.

**Úteis para investigar:**

- Acesso não autorizado: logins falhos, escalonamento de privilégio, acesso incomum a arquivos/sistemas.
- Ameaças internas (insider threats) ou atividade não autorizada: comportamentos que violam política, intencionais ou não.

**Exemplo — Windows Security Log, Event ID 4740 (conta bloqueada):**

```
Subject:
  Security ID: SYSTEM
  Account Name: WIN-R9H529RIO4Y$
  Account Domain: WORKGROUP

Account That Was Locked Out:
  Security ID: WIN-1234529ABC4Y\John
  Account Name: John

Additional Information:
  Caller Computer Name: WIN-1234529ABC4Y
```

- Mostra que a conta "John" foi bloqueada e de qual máquina a tentativa partiu.
- Pode ser usuário digitando senha errada repetidamente — ou algo mais grave. Requer logs adicionais para confirmar.

---

### 4.5 Logs de Servidores e Aplicações

- Permitem buscar comportamentos anômalos: comunicação estranha, criação de processos, escalonamento de privilégio, atividade de contas, etc.

**Exemplo — log de servidor Linux (criação de usuário):**

```
Dec 19 15:48:39 BUSDEV-007 useradd[90]: new user: name=usr1,
UID=90, GID=90, home=/home/usr1, shell=/bin/false
```

- Novo usuário `usr1` criado no servidor `BUSDEV-007`.
- `shell=/bin/false` → sem acesso a shell interativo.
- Útil para detectar criação de usuários fora do padrão da empresa ou com privilégios elevados — possível sinal de comprometimento (atacantes criam contas para manter persistência).

---

## 5. Estudo de caso: Arquitetura da Frothly Brewery

Cliente de exemplo do Wonderland SOC. O Splunk Cloud da Frothly é o ponto central de coleta, repassando os dados ao Wonderland SOC.

**Source type**: campo padrão do Splunk que identifica a estrutura de dados de um evento.

### 5.1 Ambiente Cloud — AWS

|Fonte|O que fornece|
|---|---|
|**S3 Access Logs**|Quem acessou recursos do bucket, quais ações, quando. Útil p/ detectar upload de código malicioso com credenciais comprometidas.|
|**ELB (Elastic Load Balancer)**|Logs de requisições recebidas; úteis combinados a outras fontes.|
|**EC2 + CloudWatch**|Métricas de performance das instâncias atrás do ELB. Anomalias (CPU alta, tráfego de saída incomum) podem indicar malware ou atacante presente.|
|**VPC Flow Logs**|Amostragem de tráfego (similar ao NetFlow). Útil para anomalias de padrão de tráfego, exfiltração de dados, protocolos incomuns, scanning de rede.|
|**AWS CloudTrail**|Registra chamadas de API: gerenciamento de recursos, mudanças de configuração, tentativas de acesso. Cobre login no console e chamadas programáticas. Essencial para rastrear quem/o quê/quando/onde.|

### 5.2 Ambiente Cloud — Azure

- **Active Directory** (Azure) para identidade.
- **Microsoft O365** para colaboração.
- **Sign-in activity**: dados de autenticação — indicam locais de login suspeitos, ataques de força bruta, tentativas de bypass de MFA, comportamento anômalo de login.

### 5.3 Ambiente On-Premises (sede em San Francisco)

|Fonte|Origem|O que fornece|
|---|---|---|
|**Windows Event Logs** (especialmente Security)|Workstations|Eventos de segurança do Windows|
|**OSquery logs**|Workstations Mac/Linux|Analytics do sistema operacional|
|**Symantec Antivirus logs**|Workstations|Úteis em surtos de malware ou infecções recorrentes|
|**Linux secure**|Servidores Linux|Informações de autenticação|
|**Linux_audit**|Servidores Linux|Mudanças de segurança no servidor|
|**access_combined**|Servidores web (Apache e outros não-Microsoft)|Informações de requisições HTTP|
|**Cisco ASA (firewall)**|Segmento de rede|Conexões entre segmentos internos; tentativas de conexão bloqueadas de curta duração podem indicar movimento lateral ou C2 (Command and Control) entre hosts comprometidos|

---

## 6. Principais conclusões (Takeaways)

- O Wonderland SOC é um MSSP que atende várias empresas, cada uma com arquitetura própria.
- Analistas usam teorias de defesa (Defense in Depth, Zero Trust, Active Defense, Resilience Theory) — frequentemente combinadas.
- Splunk ES é a ferramenta SIEM central; correlation searches geram notable events.
- Apps (com GUI) x Add-ons (sem GUI, coleta/otimização) — disponíveis no Splunkbase.
- Fontes de dados essenciais: tráfego de rede (pcap, NetFlow, VPC/NSG Flow Logs), IDS/IPS, firewall, endpoint, servidor/aplicação.
- DPI é caro e tráfego criptografado exige interceptação TLS (custosa e arriscada).
- Cada tipo de log tem campos e sinais específicos de investigação — é preciso desenvolver a habilidade de reconhecer informação relevante mesmo em logs desconhecidos.
- Estudo de caso Frothly Brewery ilustra como mapear fontes de dados a partir da arquitetura (cloud: AWS + Azure; on-premises: workstations, servidores, firewall).
- **Lição-chave**: nenhum analista conhece de antemão todos os detalhes de um novo ambiente — o primeiro passo é sempre entender a arquitetura para saber quais dados esperar e onde encontrá-los.