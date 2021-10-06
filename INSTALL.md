# Instalação

Este documento descreve como configurar manualmente o seu sistema para executar o OpenStreetMap Carto. Se você preferir uma configuração rápida e independente de plataforma para um ambiente de desenvolvimento, sem a necessidade de instalar e configurar ferramentas manualmente, siga um guia de instalação do Docker em [DOCKER.md](./DOCKER.md).

## OpenStreetMap data

Você precisa de dados do OpenStreetMap carregados em um banco de dados PostGIS (veja abaixo para [dependencies](#dependencies)). Estas folhas de estilo esperam um banco de dados gerado com osm2pgsql usando o backend pgsql (nomes de tabelas de `planet_osm_point`, etc), o nome do banco de dados padrão (`gis`), and the [lua transforms](https://github.com/openstreetmap/osm2pgsql/blob/master/docs/lua.md) documentado nas instruções abaixo.

Comece criando um banco de dados

```sh
sudo -u postgres createuser -s $USER
createdb gis
```

Habilite PostGIS e extensões hstore com

```sh
psql -d gis -c 'CREATE EXTENSION postgis; CREATE EXTENSION hstore;'
```

em seguida, pegue alguns dados OSM. Provavelmente é mais fácil obter um PBF de dados OSM de [Geofabrik](https://download.geofabrik.de/). Depois de fazer isso, importe com osm2pgsql:

```sh
osm2pgsql -G --hstore --style openstreetmap-carto.style --tag-transform-script openstreetmap-carto.lua -d gis ~/path/to/data.osm.pbf
```

Você pode encontrar um guia mais detalhado para configurar um banco de dados e carregar dados com osm2pgsql em [switch2osm.org](https://switch2osm.org/manually-building-a-tile-server-16-04-2-lts/).

### Índices personalizados

Os índices personalizados são necessários para o desempenho de renderização e são essenciais em bancos de dados planetários completos.

```sh
psql -d gis -f indexes.sql
```

## Download com script

Alguns recursos são renderizados usando shapefiles pré-processados.

Para baixá-los e importá-los para o banco de dados, você pode executar o seguinte script

```sh
scripts/get-external-data.py
```

O script baixa shapefiles, carrega-os no banco de dados e configura as tabelas para renderização. A documentação adicional da opção de script pode ser vista com `scripts/get-external-data.py --help`.

## Fonts

A folha de estilo usa Noto, uma família de fontes do Google com licença aberta e suporte para vários scripts. A folha de estilo usa o estilo "Sans" de Noto quando disponível. Se não estiver disponível, esta folha de estilo usa outro estilo apropriado da família Noto. A versão "UI" é usada quando disponível, com suas métricas verticais que se adaptam melhor ao texto latino.

DejaVu Sans é usada como uma fonte alternativa opcional para sistemas sem Noto Sans. Se todas as fontes Noto estiverem instaladas, ela nunca deve ser usada. A IU Noto Naskh Arabic é usada como uma fonte alternativa opcional para sistemas sem Noto Sans Arabic.

Hanazono é usado como substituto para caracteres CJK raramente usados que não são cobertos por Noto.

Unifont é usado como um recurso de último recurso, com sua excelente cobertura, presença comum nas máquinas e aparência feia. Por motivos de compatibilidade, oferecemos suporte a duas versões específicas de distribuições Linux do Unifont, portanto, espera-se que você *sempre* receba um aviso sobre a falta de uma versão Unifont.

Se você não instalar todas as fontes, a renderização em si não será interrompida, mas os glifos ausentes serão feios.

Para obter mais detalhes, consulte a documentação em [fonts.mss](style/fonts.mss).

### Instalação no Ubuntu / Debian

No Ubuntu 16.04 ou Debian Testing, você pode baixar e instalar a maioria das fontes necessárias

```sh
sudo apt-get install fonts-noto-cjk fonts-noto-hinted fonts-noto-unhinted fonts-hanazono ttf-unifont
```

Noto Emoji Regular (*not* Noto Color Emoji) pode ser baixado [from the Noto Emoji repository](https://github.com/googlei18n/noto-emoji).

Pode ser útil ter uma versão mais recente das fontes para [rare non-latin scripts](#non-latin-scripts). A versão atual da fonte upstream também tem mais alguns scripts e variantes de estilo do que no pacote do Ubuntu. Pode ser instalado [from source](https://github.com/googlei18n/noto-fonts/blob/master/FAQ.md#where-are-the-fonts).

DejaVu é empacotado como`fonts-dejavu-core`.

### Instalação em outros sistemas operacionais

As fontes podem ser baixadas aqui:

* [Noto homepage](https://www.google.com/get/noto/) e [Noto github repositories](https://github.com/googlei18n?utf8=%E2%9C%93&q=noto)
* [DejaVu homepage](http://dejavu-fonts.org/)
* [Hanazono homepage](http://fonts.jp/hanazono/)
* [Unifont homepage](http://unifoundry.com/)

Após o download, você deve instalar os arquivos de fontes da maneira usual de seu sistema operacional.

### Scripts não latinos

Para a renderização adequada de scripts não latinos, particularmente aqueles com diacríticos complicados e marcas de tom, os requisitos são

* FreeType 2.6.2 ou posterior para caracteres CJK

* Uma versão recente do Noto com cobertura para os scripts necessários.

## Dependências

Para o desenvolvimento, é necessário um estúdio de design de estilo.

* [Kosmtik](https://github.com/kosmtik/kosmtik) - Kosmtik pode ser lançado com `node index.js serve path/to/openstreetmap-carto/project.mml`

[TileMill](https://tilemill-project.github.io/tilemill/) não é oficialmente suportado, mas você pode usar uma versão recente do TileMill copiando ou vinculando simbolicamente o projeto diretamente em seu Mapbox / diretório de projeto.

Para exibir qualquer mapa, um banco de dados contendo dados do OpenStreetMap e alguns utilitários são necessários

* [PostgreSQL](https://www.postgresql.org/)
* [PostGIS](https://postgis.net/)
* [osm2pgsql](https://github.com/openstreetmap/osm2pgsql#installing) to [import your data](https://switch2osm.org/loading-osm-data/) em um banco de dados PostGIS
* Python 3 com as bibliotecas psycopg2, yaml e requisições (pacotes `python3-psycopg2`` python3-yaml` `python3-requisições` em sistemas derivados do Debian)
* `ogr2ogr` para carregar arquivos de forma no banco de dados (`gdal-bin` em sistemas derivados do Debian)

### Dependências de desenvolvimento opcionais

Algumas cores, SVGs e outros arquivos são gerados com scripts auxiliares. Nem todos os usuários precisarão dessas dependências

* Python e Ruby para executar scripts auxiliares
* [Color Math] (<https://github.com/gtaylor/python-colormath>) e [numpy] (<http://www.numpy.org/>) se estiver executando o script auxiliar generate_road_colors.py (pode ser obtido com `pip instale o colormath numpy`)

### Dependências de implantação adicionais

Para implantação, são necessários CartoCSS e Mapnik.

* [CartoCSS] (<https://github.com/mapbox/carto>)> = 0,18.0 (estamos usando YAML)
* [Mapnik] (<https://github.com/mapnik/mapnik/wiki/Mapnik-Installation>)> = 3.0

Com CartoCSS você compila essas fontes em um arquivo XML compatível com Mapnik. Ao executar o CartoCSS, especifique a versão da API Mapnik que você está usando (pelo menos 3.0.0: `carto -a" 3.0.0 "`).

Se você estiver chamando Mapnik em seu próprio programa, lembre-se de carregar o arquivo XML no modo não estrito. Dessa forma, as fontes declaradas com nomes alternativos gerarão apenas avisos, não erros. Por exemplo, usando as ligações Python, isso se torna:

```python
mapnik.load_map(mapnik.Map(width, height), xml_filename, False)  # False for non-strict mode
```
