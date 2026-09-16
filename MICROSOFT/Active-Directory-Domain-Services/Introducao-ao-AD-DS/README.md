# AD DS — Anotações de Estudo

## 1. O que é o AD DS

* Base das redes corporativas Windows.
* Banco de dados central de objetos: contas de usuário, computador, grupos.
* Diretório hierárquico e pesquisável, aplica configurações e segurança.
* Usado para: instalar/configurar apps, gerenciar segurança, acesso remoto/DirectAccess, certificados digitais.

***

## 2. Componentes Lógicos

| Componente            | Descrição                                                      |
| --------------------- | -------------------------------------------------------------- |
| **Partição**          | Parte do banco Ntds.dit (esquema, configuração, domínio)       |
| **Esquema**           | Definições dos tipos de objeto e atributos                     |
| **Domínio**           | Contêiner administrativo lógico (usuários, computadores)       |
| **Árvore de domínio** | Coleção hierárquica com namespace DNS contíguo                 |
| **Floresta**          | Um ou mais domínios com raiz, esquema e catálogo global comuns |
| **UO**                | Contêiner para delegação admin + vínculo de GPOs               |
| **Contêiner**         | Estrutura organizacional; **não aceita GPO**                   |

***

## 3. Componentes Físicos

| Componente                      | Descrição                                                                               |
| ------------------------------- | --------------------------------------------------------------------------------------- |
| **Controlador de domínio (DC)** | Guarda cópia do banco AD DS; processa e replica alterações                              |
| **Armazenamento de dados**      | Ntds.dit + logs, em `C:\Windows\NTDS`                                                   |
| **Servidor de catálogo global** | Cópia parcial/read-only de todos objetos da floresta (acelera buscas)                   |
| **RODC**                        | Controlador de domínio **somente leitura** — útil em filiais com pouca segurança física |
| **Site**                        | Contêiner de objetos ligados a um local físico                                          |
| **Sub-rede**                    | Parte dos IPs de um site (um site pode ter várias sub-redes)                            |

***

## 4. Usuários, Grupos e Computadores

### Contas de usuário

* Contêm: nome, senha, associações de grupo.
* Ferramentas de criação: **Centro Administrativo AD**, **Usuários e Computadores AD**, **Windows Admin Center**, **PowerShell**, `dsadd`.

### Contas de serviço

| Tipo                                  | Uso                                                                                                                                                                                            |
| ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Conta de serviço gerenciada (MSA)** | Gerenciamento simplificado de senha e SPN, 1 servidor                                                                                                                                          |
| **gMSA (Grupo)**                      | Mesma conta em **vários servidores** (farms, NLB, IIS). Requer chave raiz KDS: `Add-KdsRootKey –EffectiveImmediately` + `New-ADServiceAccount ... -PrincipalsAllowedToRetrieveManagedPassword` |
| **dMSA (Delegada — Server 2025)**     | Autenticação vinculada à **identidade do dispositivo**; evita roubo de credenciais; usa Credential Guard                                                                                       |

### Objetos de grupo

**Tipos:**

* **Segurança** → atribui permissões (ACLs).
* **Distribuição** → usado por apps de e-mail (sem função de segurança).

**Escopos:**

| Escopo               | Permissões em                       | Membros de                 |
| -------------------- | ----------------------------------- | -------------------------- |
| **Local**            | Só computador local                 | Qualquer lugar da floresta |
| **Local de domínio** | Todos computadores do domínio local | Qualquer lugar da floresta |
| **Global**           | Qualquer lugar da floresta          | Só do domínio local        |
| **Universal**        | Qualquer lugar da floresta          | Qualquer lugar da floresta |

### Objetos de computador

* São entidades de segurança (senha própria, se autenticam, entram em grupos).
* **Contêiner Computers**: local padrão ao ingressar no domínio — **não é UO** (não aceita GPO nem subdivisão). Recomenda-se criar UOs próprias.

***

## 5. Florestas e Domínios

### Floresta

* Container de nível mais alto; domínios compartilham **raiz, esquema e catálogo global**.

* É **limite de segurança** (nada de fora acessa por padrão) e **limite de replicação**.

* Objetos exclusivos do domínio raiz da floresta:

  * Mestre de esquema
  * Mestre de nomeação de domínio
  * Grupo Administradores de Empresa
  * Grupo Administradores de Esquema

### Domínio

* Contêiner lógico de usuários/computadores/grupos.
* **Limite de replicação** e **unidade administrativa**.
* Suporta quase 2 bilhões de objetos → maioria das orgs usa 1 domínio só.
* Funções por domínio: Mestre de RID, Mestre de infraestrutura, Emulador PDC, Grupo Administradores de Domínio.
* Fornece **autenticação** (verifica identidade) e **autorização** (controla acesso).

### Relações de confiança

| Tipo                         | Transitiva? | Direção                                             |
| ---------------------------- | ----------- | --------------------------------------------------- |
| Pai e filho                  | Sim         | Bidirecional                                        |
| Raiz de árvore               | Sim         | Bidirecional                                        |
| Externa                      | Não         | Uni ou bidirecional                                 |
| Reino (Kerberos v5)          | Sim ou não  | Uni ou bidirecional                                 |
| Floresta (completa/seletiva) | Sim         | Uni ou bidirecional                                 |
| Atalho                       | Não         | Uni ou bidirecional (manual, não existe por padrão) |

> Confianças automáticas na floresta são **transitivas**: se A confia em B e B confia em C → A confia em C.

***

## 6. Unidades Organizacionais (UOs)

**Para que servem:**

1. Consolidar objetos e aplicar **GPOs** em conjunto.
2. **Delegar** controle administrativo.

**Ferramentas de criação:** Centro Administrativo AD, Usuários e Computadores AD, Windows Admin Center, PowerShell.

### Contêineres genéricos (padrão, visíveis)

* Domínio, Contêiner interno (Builtin), Computers, Foreign Security Principals, Managed Service Accounts, Users, **UO Domain Controllers** (única UO criada por padrão).

### Contêineres ocultos (Recursos Avançados)

* LostAndFound, Program Data, System, NTDS Quotas, TPM Devices.

⚠️ **Contêineres não aceitam GPO** — só UOs aceitam.

### Design hierárquico

* Baseado em critério geográfico, funcional, de recursos ou usuários.
* Pode aninhar UOs dentro de UOs.
* Limite técnico: até 10 níveis — **recomendado: até 5 níveis**.

***

## 7. Ferramentas de Gerenciamento

| Ferramenta                           | Descrição                                                                                                                       |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| **Centro Administrativo do AD**      | GUI baseada em PowerShell; substitui "Usuários e Computadores"; gerencia UOs, políticas de senha refinadas, Lixeira do AD, etc. |
| **Windows Admin Center**             | Console web; portas padrão TCP 6516 (Win10) / TCP 443 (Server); **não instalar em um DC**                                       |
| **RSAT**                             | Ferramentas remotas de administração; habilitar via "Gerenciar recursos opcionais"                                              |
| **Módulo AD do PowerShell**          | Base de cmdlets usada por outras ferramentas                                                                                    |
| **Usuários e Computadores do AD**    | Snap-in MMC clássico (sendo substituído)                                                                                        |
| **Sites e Serviços do AD**           | Gerencia replicação e topologia de rede                                                                                         |
| **Domínios e Relações de Confiança** | Configura trusts e níveis funcionais                                                                                            |
| **Esquema do AD**                    | Edita classes/atributos — não registrado por padrão                                                                             |

***

## 🎯 Pontos-chave para prova

* Cmdlet para **criar usuário**: `New-ADUser`
* UO ≠ Contêiner → só **UO** aceita GPO.
* gMSA precisa de **chave raiz KDS**.
* Floresta = limite de segurança + replicação (esquema/config/catálogo global).
* Domínio = limite de replicação + unidade administrativa.
* RODC = ideal para filiais com segurança física fraca.
* Grupo **Universal** = mais flexível (permissões e membros em toda a floresta).
