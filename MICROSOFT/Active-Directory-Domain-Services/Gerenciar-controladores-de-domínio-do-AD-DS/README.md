
## 1. Implantação dos controladores de domínio

### Conceito principal

* Os controladores de domínio (DCs) autenticam usuários e computadores em um domínio.

* É importante planejar a quantidade e o posicionamento dos DCs, especialmente em ambientes distribuídos e de grande escala.

### 1.1 Etapas de implantação do DC

A implantação possui duas etapas:

1. Instalar os binários da função AD DS

   * Ferramentas: Windows Admin Center ou Gerenciador do Servidor.

   * Instala os arquivos necessários do Active Directory Domain Services.

   * A função ainda precisa ser configurada após a instalação.

2. Configurar o AD DS

   * Utilizar o Assistente de Configuração do Active Directory Domain Services.

   * Acessível pelo link do AD DS no Gerenciador do Servidor.

### 1.2 Perguntas da configuração do AD DS

|\
Configuração

|

O que determina

|\
\| --- | --- |\
|

Nova floresta, árvore ou DC adicional?

|

Define as informações necessárias, como o nome do domínio pai.

|\
|

Nome DNS do domínio

|

No primeiro DC, especifica-se o FQDN. Em um domínio existente, utiliza-se o nome do domínio atual.

|\
|

Nível funcional da floresta

|

Recursos disponíveis na floresta e sistemas operacionais suportados pelos DCs.

|\
|

Nível funcional do domínio

|

Recursos disponíveis no domínio e sistemas operacionais compatíveis.

|\
|

Servidor DNS

|

O DNS pode ser instalado como parte da implantação do DC.

|\
|

Catálogo global (GC)

|

Opção selecionada por padrão.

|\
|

RODC

|

Controlador de domínio somente leitura. Não está disponível para o primeiro DC da floresta.

|\
|

Senha do DSRM

|

Necessária para restaurar o banco de dados do AD DS a partir de um backup.

|\
|

Nome NetBIOS

|

Definido ao criar o primeiro controlador de domínio.

|\
|

Localização dos arquivos

|

Define onde ficam o banco de dados, os logs e o SYSVOL.

|

Diretórios padrão:

```
C:\Windows\NTDS
```

Banco de dados e arquivos de log.

```
C:\Windows\SYSVOL
```

Pasta SYSVOL.

## 2. Instalação em Server Core

O Windows Server com instalação Server Core não possui a interface gráfica do Gerenciador do Servidor.

### Métodos de instalação

* Windows Admin Center.

* Gerenciador do Servidor.

* Windows PowerShell.

* RSAT (Remote Server Administration Tools).

O RSAT pode ser instalado em versões compatíveis do Windows Server com experiência da área de trabalho ou em clientes Windows suportados, como o Windows 10.

## 3. Instalação de controlador de domínio por mídia

### Quando utilizar?

Em ambientes com conexão entre sites:

* Lenta.

* Não confiável.

* Com custo elevado.

### Objetivo

Reduzir o tráfego transferido pelo link WAN durante a implantação de um novo controlador de domínio.

### Processo

1. Criar um backup do AD DS, por exemplo, em uma unidade USB.

2. Transportar o backup até o local remoto.

3. Iniciar a instalação do AD DS no servidor remoto.

4. Selecionar a opção Instalar na mídia.

5. A maior parte da cópia dos dados ocorre localmente.

O link WAN será utilizado principalmente para:

* Tráfego relacionado à segurança.

* Alterações realizadas no AD DS após a criação da mídia.

* Sincronização das alterações feitas no domínio central.

## 4. Considerações sobre filiais (Branch Offices)

Em filiais que não possuem segurança física adequada, deve-se reduzir o impacto de uma possível violação do controlador de domínio.

### RODC — Read-Only Domain Controller

Um RODC é um controlador de domínio somente leitura.

Características:

* Contém uma cópia somente leitura do banco de dados do AD DS.

* Por padrão, não armazena senhas de usuários em cache.

* Pode ser configurado para armazenar senhas específicas de usuários da filial.

* Reduz o potencial de exposição de informações caso o controlador seja comprometido.

Aplicação prática: ambientes remotos onde o controle físico e a segurança do servidor são mais difíceis de garantir.

## 5. Atualização dos controladores de domínio

O material aborda a atualização para o Windows Server 2025 a partir do Windows Server 2012 R2 ou posterior.

### Métodos

1. Atualizar o sistema operacional dos controladores existentes.

2. Adicionar novos servidores Windows Server 2025 como controladores de domínio em um domínio existente.

### Recomendação apresentada

Adicionar novos servidores Windows Server 2025 é o método recomendado no material, pois permite uma instalação limpa do sistema operacional e do banco de dados do AD DS.

Atualização automática do DNS:

Ao adicionar um novo controlador de domínio, o Windows Server atualiza automaticamente os registros DNS do domínio, permitindo que os clientes localizem e utilizem o novo DC.

# 6. Implantação do AD DS no Azure

O Azure fornece IaaS (Infraestrutura como Serviço), permitindo executar controladores de domínio em máquinas virtuais.

Ao implantar AD DS no Azure, as regras de virtualização de controladores de domínio também devem ser consideradas.

## 6.1 Considerações principais

### Topologia de rede

* Criar uma rede virtual do Azure.

* Conectar as VMs à rede virtual.

* Para integração com um ambiente local, utilizar conectividade híbrida:

  * VPN.

  * Azure ExpressRoute.

A escolha depende de requisitos de velocidade, confiabilidade e segurança.

### Topologia do site

\- Criar e configurar um site do AD DS correspondente ao espaço de endereçamento IP da rede virtual do Azure.

### Endereçamento IP

* As VMs do Azure recebem endereços DHCP por padrão.

* É possível configurar endereços estáticos que persistem após reinicializações e desligamentos.

### DNS

O DNS interno do Azure não atende a todos os requisitos do AD DS, como registros de recursos DNS e registros SRV dinâmicos.

Alternativas:

* Função DNS do Windows Server.

* Outras soluções de DNS disponíveis no Azure, como zonas DNS privadas.

### Discos

Ao instalar AD DS em uma VM do Azure:

* Colocar `NTDS.DIT` e `SYSVOL` em um disco de dados.

* Configurar a Preferência de Cache do Host desse disco como `NONE`.

# 7. Manutenção dos controladores de domínio

## Objetivo principal

Garantir a continuidade dos serviços de autenticação e a recuperação dos dados do Active Directory.

Principais áreas:

* Disponibilidade dos controladores de domínio.

* Backup do AD DS.

* Restauração do AD DS.

* Recuperação de objetos excluídos.

## 8. Disponibilidade dos controladores de domínio

### Replicação multimaster

Os controladores de domínio utilizam um processo de replicação de vários mestres para copiar dados de um DC para outro.

Isso permite que os dados do diretório sejam replicados entre controladores.

### Recomendação de disponibilidade

O material recomenda:

* Pelo menos dois controladores de domínio por site do AD DS.

* Como mínimo absoluto para a maioria das empresas, considerar dois DCs por região geográfica.

### Benefícios

* Maior disponibilidade do banco de dados do AD DS.

* Distribuição da carga de autenticação.

* Maior resiliência durante períodos de pico de logon.

# 9. Backup e restauração do AD DS

Backups regulares são importantes, mas é fundamental saber restaurar os dados após uma falha.

## 9.1 Active Directory Recycle Bin

A Lixeira do Active Directory permite restaurar objetos excluídos sem o tempo de inatividade associado aos métodos tradicionais de restauração.

### Características

* Deve ser habilitada antes da utilização.

* Objetos excluídos ficam em um contêiner específico.

* O contêiner é exibido no Centro Administrativo do Active Directory.

* O tempo de vida padrão para novas implantações é de 180 dias, conforme o material.

* É possível restaurar objetos para o local original ou para um local alternativo.

### Limitação importante

> A Lixeira do Active Directory não pode ser usada para reverter alterações em objetos existentes.

Para reverter alterações em objetos, devem ser utilizados métodos tradicionais de backup e restauração do AD DS.

## 9.2 Backup do estado do sistema

Para restaurar o AD DS, o backup deve incluir explicitamente os dados de estado do sistema (System State).

### O que o estado do sistema inclui?

* Arquivos críticos do sistema operacional.

* Arquivos críticos das funções do servidor.

* Banco de dados do AD DS.

* Registro do Windows.

Atenção: um backup completo do servidor utilizado para recuperação total não é compatível com esse cenário específico de restauração do AD DS, conforme indicado no material.

## 9.3 DSRM — Directory Services Restore Mode

O Modo de Restauração dos Serviços de Diretório (DSRM) permite acesso aos arquivos do controlador de domínio para realizar a restauração.

### Processo geral

1. Reiniciar o controlador de domínio no modo DSRM.

2. Entrar como administrador utilizando a senha do DSRM.

3. Utilizar o Windows Server Backup para restaurar o banco de dados do diretório.

4. Reiniciar o servidor após a restauração.

5. O controlador sincroniza as alterações posteriores ao backup por meio da replicação com os parceiros.

# 10. Tipos de restauração do AD DS

## 10.1 Restauração não autoritativa

### Conceito

Restaura o controlador de domínio a partir de um backup válido conhecido, revertendo-o para um estado anterior.

Após a reinicialização:

* O DC entra em contato com seus parceiros de replicação.

* Solicita as atualizações posteriores ao backup.

* Sincroniza os dados utilizando os mecanismos normais de replicação.

### Quando utilizar?

Quando o diretório de um controlador de domínio foi danificado ou corrompido, mas o problema não se espalhou para os demais DCs.

### Limitação

Se um objeto foi excluído e essa exclusão já foi replicada para outros controladores, a restauração não autoritativa não recuperará o objeto.

Exemplo:

1. O objeto existe no backup.

2. O objeto é excluído depois do backup.

3. A exclusão é replicada para outros DCs.

4. Você restaura o backup.

5. A exclusão é replicada novamente para o controlador restaurado.

## 10.2 Restauração autoritativa

### Conceito

Permite restaurar uma cópia válida conhecida de objetos do AD DS, fazendo com que a versão restaurada substitua a versão atual durante a replicação.

### Processo geral

1. Iniciar com as etapas da restauração não autoritativa.

2. Antes de reiniciar o controlador de domínio, marcar os objetos desejados como autoritativos.

3. Após a recuperação, os objetos marcados são replicados do controlador restaurado para os parceiros de replicação.

### Diferença fundamental

|\
Não autoritativa

|

Autoritativa

|\
\| --- | --- |\
|

Recupera o DC e recebe atualizações dos parceiros.

|

Permite que os objetos marcados sejam replicados como versão válida.

|\
|

Alterações dos parceiros podem sobrescrever os dados restaurados.

|

Objetos marcados como autoritativos substituem as versões atuais na replicação.

|

# 11. Catálogo Global (Global Catalog)

## 11.1 Conceito

O Catálogo Global (GC) é uma cópia parcial, somente leitura e pesquisável de todos os objetos de uma floresta.

### Função principal

Acelerar pesquisas de objetos que podem estar armazenados em controladores de domínio diferentes dentro da floresta.

## 11.2 Catálogo Global em ambientes com múltiplos domínios

Em um único domínio:

* Cada controlador de domínio possui as informações completas dos objetos daquele domínio.

* Consultas direcionadas a um DC do domínio não retornam automaticamente objetos de outros domínios.

Para realizar consultas que incluam objetos de outros domínios da floresta, é necessário consultar um controlador de domínio que também seja um servidor de catálogo global.

### Informações armazenadas no GC

O catálogo global não contém todos os atributos de todos os objetos.

Ele mantém um subconjunto de atributos frequentemente utilizados em pesquisas entre domínios, como:

* `givenName`

* `displayName`

* `mail`

O conjunto de atributos replicados pode ser alterado por meio da modificação do esquema do AD DS.

## 11.3 Exemplos de utilização do GC

### Microsoft Exchange Server

Quando o Exchange recebe um e-mail:

1. Pesquisa a conta do destinatário.

2. Consulta o catálogo global.

3. Localiza o destinatário em um ambiente com vários domínios.

4. Determina como encaminhar a mensagem.

### Autenticação de usuários

Durante o logon:

* O controlador de domínio responsável pela autenticação pode consultar o catálogo global.

* A consulta permite verificar associações de grupos universais antes de autenticar o usuário.

## 11.4 Planejamento do catálogo global

### Ambiente com um único domínio

O material recomenda configurar todos os controladores de domínio para possuírem uma cópia do catálogo global.

### Florestas com vários domínios e múltiplos sites

Em alguns casos, pode ser necessário limitar a quantidade de DCs que hospedam o GC para reduzir o tráfego de replicação.

Consequência: isso pode criar dependência de conectividade com outros sites para executar consultas ao catálogo global.

### Recomendação do material

> Considere configurar todos os controladores de domínio como servidores de catálogo global, a menos que seja necessário reduzir o tráfego de replicação.

