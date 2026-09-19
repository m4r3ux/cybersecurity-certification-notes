# Digital Forensics & DFIR — Anotações de Estudo

Module 1

Digital Forensics

DFIR

Investigação

Material organizado para estudo, com conceitos, ferramentas, comandos e pontos importantes para sua formação em Blue Team e análise forense.

# Lesson 1.1 — Introduction to Digital Forensics

## 1. Objetivos da aula

Ao concluir esta lição, você deverá saber:

* Definir Digital Forensics e sua função na cibersegurança.

* Diferenciar DFIR, Incident Response e eDiscovery.

* Identificar tipos de investigação.

* Compreender as responsabilidades de um investigador forense.

## 2. O que é Digital Forensics?

Digital Forensics (Forense Digital) é o processo de identificar, preservar, analisar e apresentar evidências digitais de maneira legalmente admissível.

Utiliza métodos científicos para recuperar e examinar dados de dispositivos eletrônicos com finalidade investigativa.

### Características fundamentais

| Característica           | Significado                                                                   |
| ------------------------ | ----------------------------------------------------------------------------- |
| Metodologia científica   | Processo sistemático e tecnicamente fundamentado.                             |
| Legalmente defensável    | Procedimentos que podem ser justificados em um contexto legal.                |
| Resultados reproduzíveis | Outro examinador deve conseguir repetir o processo e verificar os resultados. |
| Integridade da evidência | Preservar os dados contra alterações indevidas.                               |

Ideia central: uma investigação forense não consiste apenas em encontrar um arquivo suspeito. É necessário demonstrar como a evidência foi obtida, preservada e analisada.

## 3. DFIR vs Incident Response vs eDiscovery

| Disciplina        | Foco                                  | Objetivo                                                             |
| ----------------- | ------------------------------------- | -------------------------------------------------------------------- |
| DFIR              | Ciclo completo de investigação        | Identificar causa-raiz, escopo e impacto de incidentes.              |
| Incident Response | Contenção e remediação ativas         | Interromper a ameaça, minimizar danos e restaurar operações.         |
| eDiscovery        | Coleta de documentos para fins legais | Coletar informações armazenadas eletronicamente (ESI) para litígios. |

### Sobreposição

* DFIR pode incluir atividades de resposta a incidentes.

* eDiscovery pode utilizar técnicas forenses para coleta de dados.

### Para memorizar

* IR: conter e recuperar.

* Forense: investigar e sustentar evidências.

* eDiscovery: coletar informações para processos legais.

## 4. Tipos de investigação

### 4.1 Criminal Investigations

Investigações criminais, geralmente conduzidas por órgãos de aplicação da lei.

Características:

* Conduzidas por autoridades policiais.

* Exigem procedimentos rigorosos de cadeia de custódia.

* Devem atender aos padrões de admissibilidade de provas em tribunal.

Exemplos:

* Crimes cibernéticos.

* Fraudes.

* Exploração infantil.

* Hacking.

### 4.2 Corporate Investigations

Investigações realizadas no contexto corporativo.

Exemplos:

* Violação de políticas internas.

* Roubo de propriedade intelectual.

* Conduta inadequada de funcionários.

* Investigação de vazamento de dados.

### 4.3 Civil Investigations

Investigações relacionadas a disputas civis e questões regulatórias.

Exemplos:

* Suporte a litígios.

* Disputas contratuais envolvendo evidências digitais.

* Fraudes de seguros.

* Auditorias de conformidade regulatória.

## 5. Papel do Forensic Investigator

### Responsabilidades principais

1. Proteger e preservar evidências.

2. Manter a cadeia de custódia.

3. Realizar análises completas.

4. Documentar todas as descobertas.

5. Apresentar resultados para públicos técnicos e não técnicos.

6. Prestar depoimento como especialista, quando necessário.

### Habilidades essenciais

* Conhecimento profundo de sistemas operacionais e sistemas de arquivos.

* Análise de protocolos de rede.

* Fundamentos de análise de malware.

* Documentação e comunicação.

* Conhecimento de estruturas legais e regulamentações.

## 6. Estudos de caso

### Caso 1 — Corporate Data Breach

Uma empresa de serviços financeiros detectou tráfego de saída incomum.

Descobertas da investigação DFIR:

1. Acesso inicial por e-mail de phishing.

2. Movimento lateral utilizando credenciais comprometidas.

3. Exfiltração de dados por tunelamento DNS.

4. Preservação de evidências por meio de dumps de memória e imagens de disco.

### Caso 2 — Insider Threat

Um funcionário era suspeito de roubar segredos comerciais.

Evidências encontradas:

* Downloads em massa para um dispositivo USB.

* Exclusão do histórico do navegador e de arquivos recentes.

* Recuperação de evidências por análise de MFT e artefatos de dispositivos USB.

* Reconstrução da linha do tempo indicando atividade fora do horário de trabalho.

### Caso 3 — Ransomware

Uma rede hospitalar foi criptografada por ransomware.

Descobertas forenses:

* Vetor inicial: exploração de vulnerabilidade em VPN.

* Escalonamento de privilégios por meio do Mimikatz.

* Exclusão de Shadow Copies antes da criptografia.

* Preservação da nota de resgate e dos artefatos de arquivos criptografados.

# Lesson 1.2 — Legal Aspects & Chain of Custody

## 1. Objetivos

* Compreender os princípios de integridade das evidências.

* Implementar procedimentos de cadeia de custódia.

* Utilizar hashing criptográfico para verificar evidências.

* Conhecer padrões de documentação.

* Referenciar ISO 27037 e NIST SP 800-86.

## 2. Princípios de integridade da evidência

A evidência digital é inerentemente frágil e pode ser alterada com facilidade.

| Princípio         | Explicação                                                                          |
| ----------------- | ----------------------------------------------------------------------------------- |
| Autenticidade     | Demonstrar que a evidência é realmente aquilo que afirma ser.                       |
| Confiabilidade    | O processo de coleta deve ser confiável.                                            |
| Completude        | Não excluir evidências de maneira seletiva.                                         |
| Não contaminação  | Evitar alterações durante coleta e análise.                                         |
| Reprodutibilidade | Outro examinador deve conseguir repetir o processo e alcançar as mesmas conclusões. |

### Regra de ouro

> Nunca trabalhe diretamente sobre a evidência original. Crie uma cópia forense e trabalhe nela.

Conceito importante: o objetivo é preservar a fonte original e minimizar a possibilidade de alterações durante o exame.

## 3. Chain of Custody — Cadeia de Custódia

A cadeia de custódia é um registro documentado que acompanha o manuseio da evidência desde sua apreensão até sua apresentação em tribunal.

### Elementos da cadeia de custódia

| Elemento | O que registrar                                                 |
| -------- | --------------------------------------------------------------- |
| Who      | Nome e função de cada pessoa que manipulou a evidência.         |
| What     | Descrição do item coletado.                                     |
| When     | Data e horário de cada ação ou transferência.                   |
| Where    | Local de armazenamento ou análise.                              |
| Why      | Motivo de cada transferência ou ação.                           |
| How      | Método utilizado para manipulação, transporte ou armazenamento. |

### Fluxo do processo

1. Identification — Identificar e etiquetar possíveis evidências.

2. Collection — Proteger e coletar fisicamente a evidência.

3. Transportation — Transportar com segurança até o laboratório forense.

4. Storage — Armazenar em ambiente seguro e com controle de acesso.

5. Analysis — Analisar somente cópias forenses.

6. Return/Disposal — Devolver ao proprietário ou descartar com segurança.

## 4. Hashing — MD5, SHA-1 e SHA-256

Funções de hash criptográfico produzem uma impressão digital de tamanho fixo dos dados.

Uma alteração nos dados normalmente resulta em um hash diferente.

### Algoritmos

| Algoritmo | Tamanho                               | Status apresentado                                                      |
| --------- | ------------------------------------- | ----------------------------------------------------------------------- |
| MD5       | 128 bits (32 caracteres hexadecimais) | Legado — vulnerabilidades de colisão conhecidas.                        |
| SHA-1     | 160 bits (40 caracteres hexadecimais) | Descontinuado para usos de segurança — ataques de colisão demonstrados. |
| SHA-256   | 256 bits (64 caracteres hexadecimais) | Recomendado no material — atualmente seguro.                            |

### Quando calcular hashes?

* Imediatamente após a aquisição da evidência, antes da análise.

* Após criar uma cópia forense.

* Antes e depois do transporte.

* Em cada transferência na cadeia de custódia.
