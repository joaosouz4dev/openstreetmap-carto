# Executando OpenStreetMap Carto com Docker

[Docker] (<https://docker.com>) é um ambiente virtualizado que executa um [_Docker demon_] (<https://docs.docker.com/engine/docker-overview>), no qual você pode executar software sem alterar seu host sistema permanentemente. Os componentes de software são executados em _contêineres_ que são fáceis de configurar e desmontar individualmente. O demônio Docker pode usar virtualização no nível do sistema operacional (Linux, Windows) ou uma máquina virtual (macOS, Windows).

Isso permite configurar um ambiente de desenvolvimento para o OpenStreetMap Carto facilmente. Especificamente, este ambiente consiste em um
Banco de dados PostgreSQL para armazenar os dados do OpenStreetMap e [Kosmtik] (<https://github.com/kosmtik/kosmtik>) para visualizar o estilo.

## Pré-requisitos

O Docker está disponível para Linux, macOS e Windows. [Instale] (<https://www.docker.com/get-docker>) o software empacotado para o seu sistema host em ordem
para poder executar contêineres Docker. Você também precisa do Docker Compose, que deve estar disponível assim que você instalar
O próprio Docker. Caso contrário, você precisa [instalar o Docker Compose manualmente] (<https://docs.docker.com/compose/install/>).

Você precisa de espaço em disco suficiente de _vários Gigabytes_. O Docker cria uma imagem de disco para sua máquina virtual que contém o sistema operacional virtualizado e os contêineres. O formato (Docker.raw, Docker.qcow2, \ *. Vhdx, etc.) depende do sistema host. Pode ser um arquivo esparso que aloca grandes quantidades de espaço em disco, mas ainda assim o tamanho físico começa com 2-3 GB para o sistema operacional virtual e aumenta para 6-7 GB quando preenchido com os contêineres necessários para o banco de dados, Kosmtik e um pequeno Região OSM. Mais 1-2 GB são necessários para arquivos de forma no repositório openstreetmap-carto / data.

## Começo rápido

Se você está ansioso para começar, aqui está uma visão geral das etapas necessárias.
Leia abaixo para obter os detalhes.

* `git clone https: // github.com / gravitystorm / openstreetmap-carto.git` para clonar o repositório openstreetmap-carto em um diretório em seu sistema host
* baixe os dados do OpenStreetMap no formato osm.pbf para um arquivo `data.osm.pbf` e coloque-os dentro do diretório openstreetmap-carto (por exemplo, alguma pequena área de [Geofabrik] (<https://download.geofabrik.de/>) )
* Se necessário, `sudo service postgresql stop` para ter certeza de que você não está executando um servidor PostgreSQL nativo que entraria em conflito com o servidor PostgreSQL do Docker.
* `docker-compose up import` para importar os dados (necessário apenas na primeira vez ou quando você altera o arquivo de dados)
* `docker-compose up kosmtik` para executar o aplicativo de visualização de estilo
* navegue até [http: // localhost: 6789] (http: // localhost: 6789) para ver a saída de Kosmtik
* Ctrl + C para parar o aplicativo de visualização de estilo
* `docker-compose stop db` para parar o contêiner de banco de dados

## Repositórios

As instruções acima clonarão o repositório principal do OpenStreetMap Carto. Para testar suas próprias alterações, você deve [fork] (<https://help.github.com/articles/fork-a-repo/>) repositório gravitystorm / openstreetmap-carto e [clonar seu fork] (https: //help.github .com / articles / cloning-a-repository /).

Este repositório OpenStreetMap Carto precisa ser um diretório compartilhado entre seu sistema host e a máquina virtual Docker. Os diretórios pessoais são compartilhados por padrão; se o seu repositório estiver em outro lugar, você precisará adicioná-lo à lista de compartilhamento do Docker (por exemplo, macOS: Docker Preferences> File Sharing; Windows: Docker Settings> Shared Drives).

## Importando dados

O OpenStreetMap Carto precisa de um banco de dados preenchido com dados de renderização para funcionar. Primeiro você precisa de um arquivo de dados para importar.
Provavelmente é mais fácil obter um PBF de dados OSM de [Geofabrik] (<https://download.geofabrik.de/>).
Depois de ter esse arquivo, coloque-o no diretório openstreetmap-carto e execute `docker-compose up import` no diretório openstreetmap-carto.
Isso inicia o contêiner PostgreSQL (baixa-o se não existir) e inicia um contêiner que executa [osm2pgsql] (<https://github.com/openstreetmap/osm2pgsql>) para importar os dados. O contêiner é criado na primeira vez que você executa esse comando, se ele não existir.
Na inicialização do contêiner, o script `scripts / docker-startup.sh` é invocado, preparando o banco de dados e ele mesmo inicia o osm2pgsql para importar os dados.

osm2pgsql tem algumas [opções de linha de comando] (<https://manpages.debian.org/testing/osm2pgsql/osm2pgsql.1.en.html>) e a importação por padrão usa um cache de RAM de 512 MB, 1 trabalhador e espera o arquivo de importação com o nome `data.osm.pbf`. Se você quiser personalizar qualquer um desses parâmetros, você deve definir as variáveis ​​de ambiente `OSM2PGSQL_CACHE` (por exemplo,`export OSM2PGSQL_CACHE = 1024` no Linux para definir o cache para 1 GB) para o cache de RAM (o valor depende da quantidade de RAM você tem disponível, quanto mais você pode usar aqui, mais rápida pode ser a importação), `OSM2PGSQL_NUMPROC` para o número de trabalhadores (isso depende do número de processadores que você tem e se o seu disco rígido é rápido o suficiente, por exemplo, é um SSD), ou `OSM2PGSQL_DATAFILE` se o seu arquivo tiver um nome diferente.

Você também pode [ajustar o PostgreSQL] (<https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server>) durante as fases de importação, com `PG_WORK_MEM` (padrão de 16 MB) e`PG_MAINTENANCE_WORK_MEM` (padrão de 256 MB), que eventualmente escreva `work_mem` e`maintenance_work_mem` no `postgresql.auto.conf` uma vez, tornando-os aplicados cada vez que o banco de dados for iniciado. Observe que, ao contrário das variáveis ​​osm2pgsql, uma vez definidas, você só pode alterá-las executando `ALTER SYSTEM` por conta própria, alterando`postgresql.auto.conf` ou remova o volume do banco de dados por `docker-compose down -v && docker- compor rm -v` e importe novamente.

Se você deseja personalizar e lembrar os valores, forneça-os durante sua primeira importação:

```sh
PG_WORK_MEM = 128 MB PG_MAINTENANCE_WORK_MEM = 2 GB \
OSM2PGSQL_CACHE = 2048 OSM2PGSQL_NUMPROC = 4 \
OSM2PGSQL_DATAFILE = taiwan.osm.pbf \
importação docker-compose up
```

As variáveis ​​serão lembradas em `.env` se você não tiver esse arquivo, e os valores no arquivo serão aplicados a menos que você os atribua manualmente.

Dependendo da sua máquina e do tamanho do extrato, a importação pode demorar um pouco. Quando terminar, você deve ter os dados necessários para renderizá-lo com o OpenStreetMap Carto.

## Renderização de teste

Depois de ter os dados necessários disponíveis, você pode iniciar o Kosmtik para produzir uma renderização de teste. Para isso, execute `docker-compose up kosmtik` no diretório openstreetmap-carto. Isso inicia um contêiner com Kosmtik e também inicia o contêiner de banco de dados PostgreSQL se ainda não estiver em execução. O contêiner Kosmtik é criado na primeira vez que você executa esse comando, se ele não existir.
Na inicialização do contêiner, o script `scripts / docker-startup.sh` é invocado, baixando os shapefiles necessários com`scripts / get-external-data.py` (se ainda não estiverem presentes). Posteriormente, dirige a Kosmtik. Se você precisar personalizar alguma coisa, poderá fazê-lo no script. O arquivo de configuração Kosmtik pode ser encontrado em `.kosmtik-config.yml`.
Se você quiser ter uma [configuração local] (<https://github.com/kosmtik/kosmtik#local-config>) para o nosso `project.mml`, você pode colocar um arquivo`localconfig.js` ou `localconfig.json` no diretório openstreetmap-carto.

Os dados do arquivo de forma baixados são de propriedade do usuário com UID 1000. Se você tiver outro id de usuário padrão em seu sistema, considere alterar a linha `USER 1000` no arquivo`Dockerfile`.

Após a conclusão da inicialização, você pode navegar para [http: // localhost: 6789] (http: // localhost: 6789) para ver a saída de Kosmtik. Ao pressionar Ctrl + C na linha de comando, você pode interromper o contêiner. O contêiner do banco de dados PostgreSQL ainda está em execução (você pode verificar com `docker ps`). Se você quiser interromper o contêiner do banco de dados também, execute `docker-compose stop db` no diretório openstreetmap-carto.

## Solução de problemas

A importação de dados requer uma quantidade substancial de RAM na máquina virtual. Se você encontrar o processo de importação (Leitura no arquivo: data.osm.pbf, Processamento) sendo _morto_ pelo demônio Docker, saindo com o código de erro 137, aumente a memória atribuída ao Docker (por exemplo, macOS: Docker Preferences / Windows: Docker Settings> Avançado> Ajuste os recursos de computação).

O Docker copia os arquivos de log da máquina virtual para o sistema host, sua [localização depende do SO host] (<https://stackoverflow.com/questions/30969435/where-is-the-docker-daemon-log>). Por exemplo. o 'console-ring' parece ser um ringbuffer do log do console, que pode ajudar a encontrar razões para os assassinatos.

Ao instalar o software nos contêineres e preencher o banco de dados, a imagem do disco da máquina virtual aumenta de tamanho, com o Docker alocando mais clusters. Quando o disco no sistema host está cheio (apenas alguns MB restantes), o Docker pode parecer travado. Observe os arquivos de log do sistema do seu sistema host para alocações com falha.

O Docker armazena sua imagem de disco por padrão nos diretórios pessoais do usuário. Se você não tiver espaço suficiente aqui, pode movê-lo para outro lugar. (Por exemplo, macOS: Docker> Preferências> Disco).
