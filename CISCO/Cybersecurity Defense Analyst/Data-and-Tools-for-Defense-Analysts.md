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
## Anotações — SPL, Data Models na Prática e Atividades de Investigação (Frothly)

## 1. Splunk Search Processing Language (SPL)

- **SPL**: conjunto de comandos de busca, funções, argumentos e cláusulas que dizem ao Splunk o que fazer com os eventos recuperados dos índices.
- Permite buscar, filtrar, modificar, manipular, inserir e apagar informação, tudo dentro da mesma linguagem.
- Quanto melhor o domínio de SPL, mais rápido e eficaz o analista se torna.

### Caso prático: possível exfiltração de dados (usuário BruceGist)

Notable event gerado pelo Splunk UBA (User Behavior Analytics) indicando comportamento suspeito.

**Passo 1 — mapear a atividade do usuário (dados do Cisco Network Visibility Module):**

```
user=BruceGist
| stats count by user dh
| sort -count
```

- `dh` = destination host.
- Resultado mostra os domínios mais visitados por Bruce. Achado suspeito: `cfl.dropboxstatic.com` no top 10 — a empresa usa "FrothDrive", não Dropbox.

**Passo 2 — confirmar se houve saída de dados:**

```
... dropbox ...
| stats count sum(bytes_out) as bytes_out by dh
| sort -bytes_out
```

- Soma os bytes de saída por host de destino, ordenado do maior para o menor.
- Resultado indica atividade real de envio de dados para o Dropbox (fora do padrão corporativo).

**Próximo passo sugerido:** olhar o **Data Loss Prevention (DLP) data model** ou logs de ferramentas de endpoint, cruzando o horário dos eventos.

> Boas práticas de investigação: manter notas detalhadas dos achados; seguir as políticas da empresa e o acordo de suporte com o SOC; os dados disponíveis variam conforme o cliente, então é preciso adaptar a abordagem de busca.

---

## 2. Comandos SPL úteis para investigação

|Comando|Uso|
|---|---|
|**transaction**|Agrupa eventos relacionados com base em um campo comum (ou conjunto de campos). Exemplo: agrupar sessões de um usuário com um serviço de nuvem. `user=BruceGist AND dh*.dropbox.com \| transaction user, bytes_out maxspan=1hr \| sort -bytes_out \| table dh bytes_out`|
|**top / rare**|Identifica os valores mais (`top`) ou menos (`rare`) frequentes. Muito usado em threat hunting para achar domínios DNS incomuns. Exemplo: `sourcetype="stream:dns" \| rare limit=10 "query{}"`|
|**first / last**|Encontra a primeira ou última ocorrência cronológica de um evento. Exemplo: primeira criação de processo associada a um evento de "execution" para um usuário.|
|**rex**|Extrai campos usando regex (named groups) ou substitui caracteres (sed expressions). Exemplo: extrair remetente/destinatário de um e-mail: `\| rex field=_raw "From: <(?<from>.*)> To: <(?<to>.*)>"`|

---

## 3. Transformando resultados: lookup e eval

### Lookup

- Permite **adicionar dados externos ao Splunk** (ex.: um CSV com indicadores conhecidos) e usá-los para enriquecer buscas.
- Pode ser: arquivo CSV importado, automático, scripted, vindo de banco de dados, ou usando o **KV store** do Splunk.
- Exemplo de uso: lista de hosts que deveriam estar offline, ou usuários com senhas comprometidas conhecidas — cruzada em tempo real com os dados já indexados.

**Sintaxe (obrigatório em negrito):**

```
lookup [local=<bool>] [update=<bool>] <lookup-table-name> ( <lookup-field> [AS <event-field>] )...
[ OUTPUT | OUTPUTNEW (<lookup-destfield> [AS <event-destfield>] )... ]
```

> Sinais de atividade maliciosa via DNS: aumento no volume de requisições, mudança no tipo de resource record, variação no tamanho da requisição (indicando codificação/ofuscação), variabilidade na frequência, nomes de domínio aleatórios, ou domínios levemente alterados (typosquatting).

### Eval

- Calcula expressões matemáticas, de string ou booleanas, escrevendo o resultado em um novo campo.
- Resultados numéricos/string são atribuídos automaticamente; booleanos precisam de `tostring()`.

**Exemplo:**

```
index=main sourcetype=access_combined
| eval error = if(status == 200, "OK", "Problem")
```

**Sintaxe:**

```
eval <field>=<expression>["," <field>=<expression>]...
```

---

## 4. Buscando com Data Models

- **Dataset**: coleção de dados com campos e restrições específicas que compõem um Data Model.
- Exemplo — **Endpoint Data Model**, dataset "Ports":
    - `dest_port`: busca endpoints escutando em determinada porta.
    - `user`: conta associada à porta em escuta.
- Outros datasets do Endpoint Data Model e seus usos:
    - **Registry** → escalonamento de privilégios ou desativação de recursos de segurança.
    - **Processes** → processos executados por um arquivo malicioso identificado ou usuário comprometido.
    - **File System** → arquivos suspeitos na rede ou atividade suspeita contra arquivos importantes.

### Data Models acelerados

- **Acceleration**: Splunk mantém um conjunto separado de arquivos de índice (summary index) com os datasets acelerados.
- Buscas ficam muito mais rápidas, pois usam valores já sumarizados em vez de recuperar eventos brutos.
- Muito usado para alimentar painéis de dashboard e relatórios sob demanda (ex.: painel "Traffic over time by action" do Splunk ES, baseado no Network Traffic Data Model).
- Buscas de Data Model tendem a ser mais complexas — usadas mais por Engenheiros/Arquitetos ao construir dashboards/correlation searches, mas é importante que o Analista entenda os Data Models e campos do ambiente.

**Comando `datamodel`:**

```
| datamodel [<data model name>] [<data model search mode>] [summariesonly=<bool>]
```

- **data model name**: sem esse parâmetro, retorna o JSON do data model (útil quando não há acesso fácil à documentação). Ex.: `| datamodel authentication`
- **search mode**: `search` (resultados como definidos) ou `flat` (remove a hierarquia dos nomes de campo).
- **summariesonly**: só se aplica a data models acelerados. `false` (padrão) retorna dados sumarizados e não sumarizados; `true` retorna só dados já sumarizados — útil para checar o que está sumarizado ou garantir eficiência da busca.

---

## 5. tstats (consultas estatísticas)

- Comando usado em **data models acelerados**, faz estatísticas sobre campos indexados em um arquivo de índice de séries temporais (**tsidx**).
- Arquivo de dados bruto + arquivo tsidx = conteúdo de um índice.
- Como o `tstats` busca em metadados indexados (não nos eventos brutos), é **mais rápido** que `stats`.

**Diferença entre os três comandos:**

|Comando|O que faz|
|---|---|
|**eval**|Cria novos campos a partir de campos existentes e uma expressão.|
|**stats**|Calcula estatísticas agregadas (média, contagem, soma) sobre o resultado de busca já retornado (eventos brutos).|
|**tstats**|Calcula estatísticas olhando apenas os metadados indexados — não os eventos brutos.|

**Exemplo (processos iniciados em endpoints, via Endpoint Data Model):**

```
| tstats summariesonly=true count from datamodel=Endpoint.Processes
where Processes.user="*" Processes.process=* Processes.parent_process=* Processes.user="*"
groupby _time span=1s Processes.process Processes.parent_process Processes.user
| `drop_dm_object_name("Processes")`
| table _time process parent_process user count
| sort + _time
```

---

## 6. Boas práticas de busca (Better Searching)

Um problema comum: buscar em **todos** os índices e sourcetypes de uma organização "só para não perder nada" — isso consome muito processamento (on-prem ou cloud) e ninguém tem recursos infinitos.

**Dicas de otimização:**

- Restringir o intervalo de tempo (time range) sempre que possível.
- Usar filtros apropriados de índice e sourcetype.
- Usar filtros e sub-searches para reduzir resultados e chegar a detalhes mais granulares.

---

## 7. Atividade 1 — Evidência na nuvem (AWS)

### Sourcetypes comuns do Add-on da AWS

|Sourcetype|O que fornece|
|---|---|
|**aws:cloudwatchlogs:vpcflowlog**|Metadados de tráfego IP (cabeçalho/protocolo, sem payload completo) entrando/saindo de interfaces de rede numa VPC. Equivalente ao NetFlow on-prem. Útil para diagnosticar regras restritivas de security group, monitorar tráfego, determinar direção do tráfego.|
|**aws:s3:accesslogs**|Volume de requisições, origem, quem fez a requisição, quais objetos foram acessados, quem fez upload. Captura ações PUT, GET, DELETE no bucket.|
|**aws:cloudtrail**|Quem, o quê, quando e onde agiu no ambiente AWS. Qualquer tentativa (com sucesso ou não) de ação contra um serviço AWS gera um evento CloudTrail.|
|**aws:config**|Dados de configuração (rede, EC2, VPC) e histórico de mudanças. Útil quando IPs ou nomes não batem durante uma investigação.|
|**aws:guardduty**|Serviço de detecção de ameaças da AWS — funciona como um IDS de nuvem, monitorando CloudTrail, VPC Flow e logs DNS, entre outros.|
|**AWS:SecurityHub**|Agrega findings de vários serviços AWS num só lugar, ajudando na correlação e contexto de investigações.|

### Caso 1: bucket S3 tornado público

**Contexto do notable event:**

- Bucket: `frothlywebcode`
- Tornado público às 13:01 do dia 20/08.

**Abordagem:**

- Fonte de dados: `sourcetype=aws:s3:accesslogs`, filtrando pelo nome do bucket e por eventos **após** o horário da mudança.
- Uso do comando `table` para visualizar campos relevantes.

**Achado:** a política AWS foi passada como argumento no comando `PUT ACL`. Foi possível identificar o IP de origem e o usuário responsável (`bstoll`) pela mudança de ACL no bucket. Próximo passo: verificar se a mudança foi legítima, um erro, ou uma conta comprometida.

### Caso 2: login suspeito no console AWS

**Contexto do notable event:**

- Usuário: `fr0thIy` (nome suspeito, não é um funcionário).
- IP de origem: `164.90.168.162`.

**Abordagem:**

- Fonte de dados: `sourcetype=aws:cloudtrail`, filtrando por IP de origem e pelo usuário `fr0thIy`.

**Achado:** o nome do usuário usa truques visuais (zero no lugar de "o", "I" maiúsculo no lugar de "L") — técnica comum de disfarce usada por atacantes. O usuário realizou diversas ações relacionadas à segurança.

> Dica prática: para montar uma timeline de atividades de um usuário suspeito no CloudTrail, é possível excluir ações de baixo interesse (List, Get, Describe) usando o operador `!=` no SPL.

---

## 8. Atividade 2 — "Through the Looking Glass" (Azure e Endpoint)

### Ambiente multi-cloud da Frothly

- Além da AWS, a Frothly tem infraestrutura no **Microsoft Azure** e usa **Office 365** para colaboração.
- Sourcetypes do Azure são explorados via add-on específico (Splunkbase + documentação no GitHub do add-on).

### Caso: credenciais de ex-funcionário reativadas

**Contexto do notable event:**

- Conta: `klagerfield@froth.ly` (ex-funcionário Kevin Lagerfeld — credenciais deveriam estar desativadas).
- IP de origem: `199.66.91.253` (login a partir do Canadá).

**Abordagem:**

- Fonte de dados: **Azure Active Directory sign-in logs** — `sourcetype=ms:aad:signin`.
- Busca inicial filtrada **apenas pelo IP** (sem filtrar pelo usuário suspeito).

**Achado:** dois usuários diferentes (Fyodor e `klagerfield@froth.ly`) fizeram login a partir do mesmo IP. Ao final da lista, aparece uma tentativa de login **falha** de Kevin Lagerfield no portal O365, a partir do IP suspeito.

> **Lição importante:** se a busca inicial tivesse filtrado também pelo usuário suspeito, o segundo usuário (Fyodor) usando o mesmo IP não teria sido percebido. Às vezes é preciso "dar um passo atrás" (zoom out) na busca para não perder o quadro completo — o equilíbrio entre restringir e generalizar demais é chave.

**O que os sign-in logs do Azure ajudam a responder:**

- Quantas tentativas de login falharam num período?
- Usuários estão logando de browsers/sistemas operacionais específicos?
- **Quem** (identidade), **como** (aplicação/cliente usado), **o quê** (recurso acessado).

### Endpoint on-premises: workstations Windows com infecções frequentes de malware

- Fonte de dados: sourcetype **WinEventLog** (ponto de partida).
- Como existem muitos tipos de log de evento do Windows, é possível refinar com `source=WinEventLog:Application`.
- O campo **SourceNames** mostra quais aplicações estão logando na seção "Application" do Windows Event Log — incluindo apps da Microsoft e de terceiros, como **Symantec Network Protection** e **Symantec Antivirus**.

> Lição de Robin: "às vezes você precisa buscar PELOS dados antes de poder buscar NOS dados" — ou seja, primeiro descobrir quais fontes de dados existem no ambiente, depois investigar de fato.

---

## 9. Revisão geral do curso (Onboarding Wonderland SOC)

|Bloco|Conteúdo coberto|
|---|---|
|**Ferramentas para Analistas**|Splunk Enterprise Security, Splunk SOAR, Wireshark/Tshark, Tcpdump, CyberChef, Splunkbase (apps e add-ons).|
|**Dados para Defesa**|Categorias de dispositivos e dados: autenticação, tráfego de rede, proxy/gateway, aplicação, endpoint, servidor; exemplos de IDS/IPS e firewall.|
|**Cyber Threat Intelligence**|Fontes internas e externas de CTI; formatos (IOCs, TTPs, blogs, relatórios); colaboração da comunidade de segurança; não achar correspondência não significa ausência de ameaça.|
|**Usando dados no Splunk**|Normalização via CIM; SPL; Data Models.|

### Objetivos de aprendizagem alcançados no curso

- Identificar tipos comuns de sistemas de defesa, ferramentas de análise e fontes de dados úteis (on-prem e cloud).
- Identificar os níveis de Threat Intelligence e sua aplicação.
- Pesquisar fontes de inteligência em busca de IOCs.
- Descrever boas práticas de SIEM e conceitos básicos do Splunk ES (CIM, Data Models, acceleration).
- Explicar e usar comandos SPL comuns: **TSTATS, TRANSACTION, FIRST/LAST, REX, EVAL, LOOKUP**.
- Aplicar boas práticas para compor buscas eficientes.
- Identificar recursos de SPL: **Splunk Security Essentials** e **Splunk Lantern**.
- Descrever como o Splunk Security Essentials pode avaliar fontes de dados ou conteúdo para um sourcetype específico.
- Analisar informações de um evento de segurança e determinar quais fontes de dados melhor guiam a investigação.

**Próximo curso da trilha:** "The Art of Investigation" (foco em praticar investigações reais com cenários e estratégias).