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

## 6. Cyber Threat Intelligence (CTI)

Introduzido por Taylor, Engenheiro de SOC (desenvolve correlation searches que geram Notable Events).

### O que é

- **Cyber Threat Intelligence**: coleta e contextualização de dados sobre indicadores, técnicas e táticas, usados para detecção, mitigação, análise e resposta a ameaças baseada em risco.
- Na forma mais simples: lista de **IOCs (Indicators of Compromise)** — objetos ligados a atividade maliciosa conhecida.

### Níveis de Threat Intelligence

|Nível|Foco|Uso principal|
|---|---|---|
|**Tática**|IOCs: URLs, IPs, hashes de arquivo, assinaturas de vírus|Identificar/confirmar ameaças ativas no ambiente. Corresponde à base da **Pyramid of Pain**. Dado "estala" (fica velho) rápido — atores mudam infraestrutura de ataque com frequência. Mais útil quando integrada direto às ferramentas de monitoramento.|
|**Operacional**|TTPs (Tactics, Techniques and Procedures)|Fica no topo da Pyramid of Pain. Ajuda a melhorar monitoramento, investigações, decisões de sistema/política e threat hunting (permite identificar rapidamente a extensão de um comprometimento). Recurso público de referência: **MITRE ATT&CK Framework** (usado dentro do Splunk ES ao investigar Notable Events).|
|**Estratégica**|Visão ampla do cenário de ameaças|Direciona a estratégia organizacional geral (mais voltada a líderes).|

> Analistas trabalham principalmente com os níveis **tático** e **operacional**.

### Fontes de Threat Intelligence

**Internas à organização:**

- Relatórios de incidentes, tickets, anotações de investigações anteriores, logs de e-mails suspeitos.
- Dados históricos únicos da empresa, revelam padrões próprios ao longo do tempo.

**Externas à organização:**

|Tipo|Características|Exemplos|
|---|---|---|
|**Sharing Groups (ISAC / ISAO)**|Organizações formadas por membros que centralizam e compartilham inteligência coletiva|**ISAC**: focado em setores de infraestrutura crítica (ex.: FS-ISAC para serviços financeiros, ISAC de Retail & Hospitality). **ISAO**: cobre indústrias/regiões semelhantes no setor público e privado (ex.: CompTIA, grupos estaduais de governo).|
|**Open Source Intelligence (OSINT)**|Blogs, feeds RSS, APIs abertas. Valioso, porém menos curado/monitorado — muitas vezes só lista IOCs, sem contexto adicional.|US-CERT, Hybrid Analysis, AlienVault|
|**Commercial Intelligence**|Fornecedores especializados, geralmente pagos (algumas versões gratuitas limitadas). Entrega dados curados, com score de reputação/risco e relatórios detalhando como a ameaça opera.|Recorded Future, CrowdStrike Falcon Intelligence, Mandiant|

> Cada fornecedor/comunidade costuma se especializar em um tipo de dado (ex.: só domínios de phishing, ou só atores de malware) — por isso é comum assinar/consultar **múltiplas fontes**.

### Por que CTI é valiosa

- Combinada com monitoramento automatizado, melhora a qualidade dos alertas e reduz falsos positivos.
- Acelera a triagem, permitindo validar rapidamente se um objeto observado está ligado a uma ameaça conhecida.
- **Dissemination (disseminação)** é essencial: compartilhar o que se aprende com outros times, dentro e fora da organização, fortalece a comunidade de defesa como um todo.

### Pontos-chave finais sobre CTI

- Diferentes fontes são especializadas em diferentes tipos de dado — vale conhecer mais de uma.
- OSINT é construída pela comunidade, nem sempre totalmente curada — use como apoio, não como única fonte de decisão.
- **Não encontrar correspondência** em uma base de Threat Intel não significa que o indicador é inofensivo — pode simplesmente não ter sido reportado/identificado ainda. Continue investigando até ter certeza.
- Objetivo final: **automatizar** o uso de CTI (integração com ferramentas) para ganhar eficiência — trabalho conjunto entre Analistas e Engenheiros.

### Ferramentas e protocolos de CTI (bom conhecer, mesmo fora da área de Engenharia)

|Ferramenta/Protocolo|Descrição|
|---|---|
|**MISP** (Malware Information Sharing Platform)|Plataforma open-source e gratuita para armazenar, compartilhar e trabalhar com threat intelligence em larga escala. Projeto comunitário.|
|**TAXII** (Trusted Automated Exchange of Intelligence Information)|Protocolo de aplicação para troca de CTI via HTTPS. Usado por ferramentas/fornecedores para operacionalizar a inteligência.|
|**STIX** (Structured Threat Information Expression)|Linguagem/formato de serialização para facilitar a troca de CTI. Open source e gratuito.|

---

## 7. Splunk na prática (com Ashley, Analista SOC I)

### Campos padrão para identificar a origem do dado

|Campo|Definição|
|---|---|
|**host**|Nome do dispositivo físico ou virtual de onde o evento se origina (ex.: endpoint Linux/Windows, firewall, proxy).|
|**source**|Nome do arquivo, diretório, data stream ou input de onde o evento se origina (ex.: caminho completo do arquivo/diretório monitorado).|
|**sourcetype**|Campo padrão que identifica a **estrutura de dados** do evento. Determina como o Splunk formata o dado na indexação e como ele é exibido nas buscas.|

> Dados de hosts/sources diferentes podem compartilhar o mesmo sourcetype. Ex.: logs Linux em `/var/log/messages` e Syslog recebido via UDP:514 de outro servidor podem ter o mesmo sourcetype `linux_syslog`.

### Comandos úteis para explorar um ambiente novo (Splunk Search & Reporting / "Search App" / "Core Splunk")

**1. Descobrir os índices (indexes)** — repositórios onde o Splunk armazena os dados indexados (em arquivos flat no indexer):

```
index=* | stats count by index | fields index
```

**2. Descobrir os hosts** — usa o comando `metadata`:

```
| metadata type=hosts index=*
```

**3. Descobrir os sourcetypes:**

```
| metadata type=sourcetypes index=*
```

**4. Descobrir os sources:**

```
| metadata type=sources index=*
```

> O comando `metadata` retorna uma lista de sources, sourcetypes ou hosts de um índice (ou peer de busca distribuída) especificado.

### Origem dos sourcetypes

- **Pretrained (pré-treinados)**: sourcetypes nativos/embutidos do Splunk, reconhecidos e atribuídos automaticamente à maioria dos dados recebidos. Também podem ser atribuídos manualmente quando o Splunk não reconhece o formato sozinho.
- **De Add-ons e Apps**: add-ons trazem inputs pré-configurados que definem o sourcetype apropriado para uma tecnologia de terceiros, podendo separar os dados em vários sourcetypes específicos.
    - Exemplo AWS: `aws:cloudwatchlogs` (dado genérico do CloudWatch Logs) vs. `aws:cloudwatchlogs:vpcflow` (sourcetype específico para VPC Flow Logs vindos do CloudWatch Logs).

### Por que a normalização de dados importa

- Incidentes de segurança raramente se limitam a um único dispositivo: atacantes tentam múltiplos pontos de entrada e, após o sucesso, tendem a se mover lateralmente pela rede.
- Analistas precisam investigar múltiplas camadas de defesa, cada uma gerando dados em formatos diferentes.
- Como ninguém pode ser especialista em todos os tipos de log, a **normalização** (seguir um esquema de classificação comum) permite trabalhar com qualquer fonte sem precisar conhecer a sintaxe exata de cada uma.

---

## 8. Splunk Data Models e o Common Information Model (CIM)

### O que são Data Models

- Um **data model** é um conjunto padronizado de nomes e valores de campo que facilita trabalhar com múltiplas fontes de dados.
- Analogia: é como rotular todas as caixas de uma gaveta bagunçada — na próxima vez que precisar de tesoura, grampeador ou fita, você sabe exatamente em qual caixa procurar, sem revirar tudo.

### Common Information Model (CIM)

- O **CIM** é uma coleção de data models definidos pelo Splunk para os tipos de dado mais relevantes à segurança em ambientes corporativos.
- A maioria das ferramentas Splunk (nativas e de terceiros no Splunkbase), assim como praticamente todas as correlation searches padrão do Enterprise Security, **esperam dados compatíveis com o CIM**.
- Adotar o CIM evita customização desnecessária e permite que relatórios, alertas e detecções funcionem "out of the box".
- Tornar os dados CIM-compliant normalmente é responsabilidade de **Engenheiros/Arquitetos**, não do Analista — mas se um relatório não estiver populando ou um alerta não disparar como esperado, a causa pode ser dado fora do padrão CIM.

### Exemplo: Data Model de Authentication

- Reúne eventos de **todas** as fontes/índices marcados com a tag "authentication" (ex.: Windows Security logs, logs de autenticação Linux, Active Directory).
- Tem hierarquia interna: autenticações falhas, autenticações bem-sucedidas, e outras categorias de mensagens, funcionando como filtros/constraints adicionais.
- Não é necessário especificar "autenticação Windows" ou "autenticação de rede" separadamente — o data model já cobre todos os tipos.
- Data models também são usados nos bastidores pelas **correlation searches** do Splunk ES.

### Comparando busca com SPL puro x busca com Data Model

**Buscando ataques de força bruta (SPL puro, só Windows):**

```
index=example sourcetype=win*security user=* user!=""
| transaction EventCode=4625
| stats count by host, user, src_ip
| where count > 50
```

Problema: cobre só Windows. Para incluir Linux, O365, Azure, AWS, a busca cresceria muito e não escalaria bem.

**Mesmo objetivo, usando o Data Model de Authentication (exemplo do Splunk ES):**

```
| from datamodel:"Authentication"."Authentication"
| stats values(tag) as tag, values(app) as app, count(eval('action'=="failure")) as failure, count(eval('action'=="success")) as success by src
| search success>0
| xswhere failure from failures_by_src_count_1h in authentication is above medium
```

Vantagem: cobre automaticamente Windows, Linux, cloud (Azure/AWS) e qualquer outra fonte normalizada com a tag de autenticação — sem precisar reescrever a lógica para cada tecnologia.

> **Lição central**: SOC teams devem se esforçar para garantir que todo dado enviado ao Splunk seja **CIM compliant**.

### Data Models comuns em investigações de segurança

Authentication, Data Access, Databases, Data Loss Prevention, Email, Endpoint, Intrusion Detection, Malware, Network Traffic, Vulnerabilities, Web.

> O(s) data model(s) mais úteis dependem do que está sendo investigado e dos dados disponíveis.

---

## 9. Principais conclusões (Takeaways)

- O Wonderland SOC é um MSSP que atende várias empresas, cada uma com arquitetura própria.
- Analistas usam teorias de defesa (Defense in Depth, Zero Trust, Active Defense, Resilience Theory) — frequentemente combinadas.
- Splunk ES é a ferramenta SIEM central; correlation searches geram notable events.
- Apps (com GUI) x Add-ons (sem GUI, coleta/otimização) — disponíveis no Splunkbase.
- Fontes de dados essenciais: tráfego de rede (pcap, NetFlow, VPC/NSG Flow Logs), IDS/IPS, firewall, endpoint, servidor/aplicação.
- DPI é caro e tráfego criptografado exige interceptação TLS (custosa e arriscada).
- Cada tipo de log tem campos e sinais específicos de investigação — é preciso desenvolver a habilidade de reconhecer informação relevante mesmo em logs desconhecidos.
- Estudo de caso Frothly Brewery ilustra como mapear fontes de dados a partir da arquitetura (cloud: AWS + Azure; on-premises: workstations, servidores, firewall).
- **Lição-chave**: nenhum analista conhece de antemão todos os detalhes de um novo ambiente — o primeiro passo é sempre entender a arquitetura para saber quais dados esperar e onde encontrá-los.
- Cyber Threat Intelligence (CTI) fornece contexto (tático, operacional, estratégico) que ajuda a validar rapidamente se algo observado está ligado a uma ameaça conhecida.
- Fontes de CTI vão de dados internos (histórico da própria empresa) a sharing groups (ISAC/ISAO), OSINT e fontes comerciais — o ideal é combinar várias.
- Splunk identifica a origem de um dado por três campos: host, source e sourcetype; comandos como `metadata` ajudam a mapear um ambiente novo rapidamente.
- O Common Information Model (CIM) normaliza dados de fontes diferentes sob nomes de campo padronizados, permitindo que data models (ex.: Authentication) substituam buscas SPL longas e não escaláveis.