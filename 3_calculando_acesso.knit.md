# Calculando acessibilidade urbana em R

## Cálculo da matriz de tempo de viagem

Como comentado anteriormente, a primeira etapa necessária para calcular os níveis de acessibilidade de uma área urbana é calcular a matriz de custo de viagem entre as diversas origens e destinos que a compõem. Na literatura científica e na prática do planejamento de sistemas de transporte público, esse custo é mais frequentemente representado pelo tempo de viagem que separa dois pontos [@el-geneidy2016cost; @venter2016assessing], embora trabalhos recentes tenham considerado também outros fatores, como o dinheiro necessário para realizar uma viagem e o nível de conforto da viagem entre um ponto e outro [@arbex2020estimating; @herszenhut2022impact]. Pela prevalência deste tipo de matriz na literatura e na prática, porém, iremos nos focar em matrizes de tempo de viagem.

Atualmente, a forma mais fácil e rápida de gerar uma matriz de tempo de viagem em R é utilizando o pacote r5r [@pereira2021r5r], desenvolvido pela equipe do Projeto Acesso a Oportunidades, do Ipea. O pacote utiliza, por trás dos panos, o *software* de roteamento multimodal de transporte público R5, desenvolvido pela Conveyal[^1].

[^1]: Disponível em <https://github.com/conveyal/r5>.

### Instalação do r5r

A instalação do `r5r` funciona como a instalação de qualquer pacote no R.


::: {.cell}

```{.r .cell-code}
install.packages("r5r")
```
:::


Além do R, o pacote `r5r` requer também a instalação do Java 11[^2]. Se você não sabe qual versão do Java você tem instalada em seu computador, você pode checar essa informação rodando este código no console do R.

[^2]: O Java 11 pode ser baixado em <https://www.oracle.com/java/technologies/downloads/#java11> ou em <https://jdk.java.net/java-se-ri/11>.


::: {.cell}

```{.r .cell-code}
cat(processx::run("java", args = "--version")$stdout)
```

::: {.cell-output .cell-output-stdout}
```
java 11.0.10 2021-01-19 LTS
Java(TM) SE Runtime Environment 18.9 (build 11.0.10+8-LTS-162)
Java HotSpot(TM) 64-Bit Server VM 18.9 (build 11.0.10+8-LTS-162, mixed mode)
```
:::
:::


### Dados necessários

O uso do pacote `r5r` requer os seguintes dados:

- **Rede viária** (obrigatório): um arquivo com a rede viária e de infraestrutura de pedestres do *OpenStreetMap*, em formato `.pbf`;
- **Rede de transporte público** (opcional): um arquivo GTFS descrevendo a rede de transporte público da área de estudo;
- **Topografia** (opcional): um arquivo de dados em *raster* com o modelo digital de elevação em formato `.tif`, caso deseje-se levar em consideração os efeitos da topografia do local sobre os tempos de caminhada.

Aqui estão alguns lugares de onde você pode baixar estes dados:

- OpenStreetMap
  - [osmextract](https://docs.ropensci.org/osmextract/), pacote de R
  - [geofabrik](https://download.geofabrik.de/), website
  - [hot export tool](https://export.hotosm.org/), website
  - [BBBike.org](https://extract.bbbike.org/), website

- GTFS
  - [tidytransit](http://tidytransit.r-transit.org/), pacote de R
  - [transitland](https://www.transit.land/), website
  - No capítulo X desde livro (tabela xx) nós indicamos também onde baixar os dados de GTFS de algumas cidades brasileiras que compartilham seus dados publicamente.

- Topografia
  - [elevatr](https://github.com/jhollist/elevatr), pacote de R
  - [Nasa's SRTMGL1](https://lpdaac.usgs.gov/products/srtmgl1v003/), website 

Os arquivos destes dados devem ser salvos em uma mesma pasta que, preferencialmente, não contenha nenhum outro arquivo. Como se verá adiante, o `r5r` combina todos os dados salvos nesta pasta para criar uma rede de transporte multimodal que será utilizada para simulações de roteamento e cálculo das matrizes de viagem. Note que é possível ter mais de um arquivo GTFS na mesma pasta, nesse caso o `r5r` irá considerar as redes de transporte públicos de todos *feeds*. No entanto, a pasta deve conter um único arquivo de rede viária `.pbf`. Assumindo que os scripts de R estarão em uma pasta chamada R, a organização dos arquivos deverá seguir o esquema abaixo:


::: {.cell}
::: {.cell-output .cell-output-stdout}
```
C:\Users\user\AppData\Local\Temp\RtmpwHgVNV/projeto_acessibilidade
+-- R
|   +-- script1.R
|   \-- script2.R
\-- r5
    +-- rede_transporte_publico.zip
    +-- rede_viaria.osm.pbf
    \-- topografia.tif
```
:::
:::


Para ilustrar as funcionalidades do `r5r`, nós vamos usar uma pequena amostra de dados para a cidade de Porto Alegre (Brasil). Esses dados estão disponíveis dentro do próprio pacote `r5r` na pasta `system.file("extdata/poa", package = "r5r")`:


::: {.cell}
::: {.cell-output .cell-output-stdout}
```
R:/Dropbox/git/intro_access_book/renv/library/R-4.1/x86_64-w64-mingw32/r5r/extdata/poa
+-- poa.zip
+-- poa_elevation.tif
+-- poa_hexgrid.csv
+-- poa_osm.pbf
\-- poa_points_of_interest.csv
```
:::
:::


Esta pasta possui quatro arquivos que vamos usar agora:

- A rede viária do OpenStreetMap: `poa_osm.pbf`
- Dois feeds de GTFS das redes de ônibus e de trens: `poa_eptc.zip` e `poa_trensurb.zip`
- O dado de topografia: `poa_elevation.tif`
- Um arquivo `poa_hexgrid.csv` com coordenadas geográficas dos centróides de uma grade hexagonal regular cobrindo toda a área da amostra e com informações sobre o tamanho da população residente e o número de oportunidades (empregos, escolas e hospitais) em cada hexágono. Esses pontos serão utilizados como os pontos de origem e destino no cálculo da matriz de tempo de viagem.

### Calculando a matriz de tempo de viagens

Antes de calcular a matriz de tempo de viagem, precisamos aumentar a memória disponível para o Java. Isto é necessário porque, por padrão, o R aloca apenas 512MB de memória para processos Java, o que não é suficiente para grandes consultas usando `r5r`. Para aumentar a memória disponível para 2GB, por exemplo, precisamos definir um valor para o parâmetro  `java.parameters` no início do script antes de carregar as bibliotecas do R:


::: {.cell}

```{.r .cell-code}
options(java.parameters = "-Xmx2G")
```
:::


Pronto, agora vamos carregar as bibliotecas que vamos utilizar neste capítulo.


::: {.cell}

```{.r .cell-code}
library(r5r)
library(accessibility)
library(sf)
library(data.table)
library(ggplot2)
library(aopdata)
```
:::


Feito isso, o cálculo de uma matriz de tempo de viagens da sua área de estudo pode ser feito em dois passos. O primeiro passo é gerar a rede de transporte multimodal que será utilizada no roteamento. Para isso, nós utilizamos a função `setup_r5()`. Esta função baixa o *software* de roteamento R5 e o utiliza para criar a rede. A função `setup_r5()` recebe como *input* o caminho da pasta onde você armazenou seus dados. Além da função salvar na pasta alguns arquivos necessários para o roteamento, ela também retorna uma conexão com o R5, que neste exemplo nós chamamos de `r5r_core`, e que será utilizada no cálculo da matriz de tempo de viagem.


::: {.cell}

```{.r .cell-code}
path <- system.file("extdata/poa", package = "r5r")

r5r_core <- setup_r5(path, use_elevation = TRUE, verbose = FALSE)

fs::dir_tree(path)
```

::: {.cell-output .cell-output-stdout}
```
R:/Dropbox/git/intro_access_book/renv/library/R-4.1/x86_64-w64-mingw32/r5r/extdata/poa
+-- network.dat
+-- poa.zip
+-- poa_elevation.tif
+-- poa_hexgrid.csv
+-- poa_osm.pbf
+-- poa_osm.pbf.mapdb
+-- poa_osm.pbf.mapdb.p
\-- poa_points_of_interest.csv
```
:::
:::


O passo final é usar a função para o cálculo da matriz de tempo de viagem, apropriadamente chamada de `travel_time_matrix()`. Como *inputs* básicos, a função recebe a conexão com o R5 criada acima, pontos de origem e destino em formato de `data.frame` com as colunas `id`, `lon` e `lat`, o modo de transporte a ser utilizado, o horário de partida, o tempo máximo de caminhada permitido da origem até o embarque no transporte público e do desembarque até o destino, e o tempo máximo de viagem a ser considerado.


::: {.cell}

```{.r .cell-code}
points <- fread(file.path(path, "poa_hexgrid.csv"))

ttm <- travel_time_matrix(
  r5r_core,
  origins = points,
  destinations = points,
  mode = c("WALK", "TRANSIT"),
  departure_datetime = as.POSIXct(
    "13-05-2019 14:00:00",
    format = "%d-%m-%Y %H:%M:%S"
  ),
  max_walk_dist = 800,
  max_trip_duration = 120,
  verbose = FALSE,
  progress = FALSE
)

ttm
```

::: {.cell-output .cell-output-stdout}
```
                  fromId            toId travel_time
      1: 89a901291abffff 89a901291abffff           1
      2: 89a901291abffff 89a901295b7ffff          41
      3: 89a901291abffff 89a9012809bffff          43
      4: 89a901291abffff 89a901285cfffff          35
      5: 89a901291abffff 89a90e934d7ffff          71
     ---                                            
1169147: 89a90166da7ffff 89a90129133ffff          58
1169148: 89a90166da7ffff 89a9012ac43ffff          93
1169149: 89a90166da7ffff 89a90129a47ffff          19
1169150: 89a90166da7ffff 89a90128883ffff          65
1169151: 89a90166da7ffff 89a90166da7ffff           0
```
:::
:::


Diversos outros *inputs* podem ser passados para o cálculo da matriz, como a velocidade de caminhada e o número máximo de pernas de transporte público permitido, entre outros. Para mais informações sobre cada um dos parâmetros, por favor consulte a documentação da função disponível no pacote ou no site do `r5r` [aqui](https://ipeagit.github.io/r5r/reference/travel_time_matrix.html).

Na prática, o que a função `travel_time_matrix()` faz é encontrar qual a rota de viagem mais rápida partindo de cada ponto de origem para todos os possíveis pontos de destino considerando o modo de viagem, horário de partida e demais parâmetros passados pelo usuário. Para isso, o `r5r` considera tempos de viagem de porta-a-porta. No caso de uma viagem por transporte público, por exemplo, o tempo total de viagem inclui a) o tempo de caminhada até a parada de transporte público; b) o tempo de espera pelo veículo na parada; c) o tempo de deslocamento dentro do veículo; e d) o tempo de viagem a pé da parada de desembarque até o destino. Em casos em que mais de uma rota de transporte público é utilizada, o `r5r` também contabiliza o tempo gasto nas conexões, considerando a caminhada entre paradas e o tempo de espera pelo próximo veículo.

::: {.callout-note appearance="simple"}
A função `travel_time_matrix()` utiliza uma extensão do algoritmo de roteamento RAPTOR [@conway2017evidencebased], o que torna o R5 extremamente rápido. A depender da quantidade de pares de origem-destino, o `r5r` pode ser entre 6 e 200 vezes mais rápido do que *softwares* alternativos para calcular uma matriz de tempo de viagem [@higgins2022calculating].
:::

## Cálculo de acessibilidade

Após calculada a matriz de tempo de viagem entre as origens e os destinos da área de estudo, nós precisamos utilizá-la para calcular os níveis de acessibilidade do local. Para isso, nós utilizaremos o pacote `accessibility`, também desenvolvido pela equipe do Projeto Acesso a Oportunidades, que contém diversas funções para vários indicadores de acessibilidade.

Como *input* básico, todas as funções requerem uma matriz de custo pré-calculada (no nosso caso, a matriz de tempo de viagem calculada na seção anterior) e dados de uso do solo, como o número de determinados tipos de oportunidades em cada ponto da área de estudo, por exemplo.

**Medida cumulativa de acesso a oportunidades**

O exemplo abaixo mostra uma simples aplicação da função `cumulative_cutoff()`, utilizada para calcular o número de oportunidades que pode ser alcançado em um determinado limite de custo de viagem. No exemplo abaixo, nós calculamos o número de postos de trabalho que podem ser alcançados em até 30 minutos de viagem a partir de cada origem presente em nossa matriz de tempo de viagem.


::: {.cell}

```{.r .cell-code}
setnames(ttm, c("fromId", "toId"), c("from_id", "to_id"))

cumulative_access <- cumulative_cutoff(
  ttm,
  points,
  opportunity = "schools",
  travel_cost = "travel_time",
  cutoff = 30
)

cumulative_access
```

::: {.cell-output .cell-output-stdout}
```
                   id schools
   1: 89a901291abffff      19
   2: 89a9012a3cfffff       0
   3: 89a901295b7ffff      15
   4: 89a901284a3ffff       0
   5: 89a9012809bffff      20
  ---                        
1223: 89a90129133ffff       4
1224: 89a9012ac43ffff       9
1225: 89a90129a47ffff      12
1226: 89a90128883ffff      31
1227: 89a90166da7ffff      15
```
:::
:::


**Mínimo custo de viagem**

A função `cost_to_closest()`, por sua vez, calcula o mínimo custo de viagem necessário para alcançar um determinado número de oportunidades. O código abaixo, por exemplo, calcula o tempo de viagem mínimo para alcançar o hospital mais próximo a partir de cada origem.


::: {.cell}

```{.r .cell-code}
min_cost <- cost_to_closest(
  ttm,
  points,
  opportunity = "schools",
  travel_cost = "travel_time"
)
```

::: {.cell-output .cell-output-stderr}
```
Warning in `[.data.table`(filled_access_df, is.na(get(access_col)),
`:=`(eval(access_col, : inf (type 'double') at RHS position 1 truncated
(precision lost) when assigning to type 'integer' (column 2 named 'min_cost')
```
:::

```{.r .cell-code}
min_cost
```

::: {.cell-output .cell-output-stdout}
```
                   id travel_time
   1: 89a9012124fffff           0
   2: 89a9012126bffff          16
   3: 89a9012127bffff          11
   4: 89a90128003ffff           7
   5: 89a90128007ffff          20
  ---                            
1223: 89a90e934cbffff          13
1224: 89a90e934cfffff           9
1225: 89a90e934d3ffff           7
1226: 89a90e934d7ffff           0
1227: 89a90e934dbffff          23
```
:::
:::


**Medidas gravitacionais de acessibilidade**

A função `gravity()` calcula medidas gravitacionais de acessibilidade - ou seja, aquelas nas quais o peso de cada oportunidade diminui gradualmente com o aumento do custo de viagem. Existem, no entanto, uma gama de diferentes tipos de funções de decaimento que podem ser utilizadas, como funções de decaimento exponenciais negativas, de potências inversas, entre outras. Por isso, esta função recebe um *input* adicional: a função de decaimento a ser utilizada no cálculo. O exemplo abaixo apresenta o cálculo de acessibilidade a estabelecimentos de educação usando uma medida gravitacional exponencial negativa com parâmetro de decaimento igual a `0.2`.


::: {.cell}

```{.r .cell-code}
negative_exp_access <- gravity(
  ttm,
  points,
  opportunity = "schools",
  travel_cost = "travel_time",
  decay_function = decay_exponential(0.2)
)

negative_exp_access
```

::: {.cell-output .cell-output-stdout}
```
                   id    schools
   1: 89a901291abffff 0.35892873
   2: 89a9012a3cfffff 0.00000000
   3: 89a901295b7ffff 0.52061269
   4: 89a901284a3ffff 0.00000000
   5: 89a9012809bffff 0.44769080
  ---                           
1223: 89a90129133ffff 0.02885676
1224: 89a9012ac43ffff 0.09138463
1225: 89a90129a47ffff 0.35857154
1226: 89a90128883ffff 1.86366449
1227: 89a90166da7ffff 0.40513107
```
:::
:::


**Indicadores de acessibilidade com competição**

Por fim, a função `floating_catchment_area()` calcula níveis de acessibilidade levando em consideração a competição por oportunidades usando diferentes indicadores do tipo *floating catchment area*. Como diversos métodos de FCA podem ser utilizados, a função requer que o método desejado seja explicitamente assinalado. E, assim como a função de acessibilidade gravitacional, a função de decaimento utilizada também deve ser definida pelo usuário. O código a seguir mostra um exemplo de cálculo de acessibilidade a oportunidades de emprego usando o método BFCA [@paez2019demand], levando em consideração os efeitos de competição entre a população como um todo e uma função de decaimento exponencial com parâmetro de decaimento igual a 2.


::: {.cell}

```{.r .cell-code}
bfca_access <- floating_catchment_area(
  ttm,
  points,
  opportunity = "schools",
  travel_cost = "travel_time",
  demand = "population",
  method = "bfca",
  decay_function = decay_exponential(0.05)
)

bfca_access
```

::: {.cell-output .cell-output-stdout}
```
                   id      schools
   1: 89a901291abffff 0.0002574442
   2: 89a9012a3cfffff 0.0000000000
   3: 89a901295b7ffff 0.0002091911
   4: 89a901284a3ffff 0.0000000000
   5: 89a9012809bffff 0.0002276971
  ---                             
1223: 89a90129133ffff 0.0001352617
1224: 89a9012ac43ffff 0.0001416331
1225: 89a90129a47ffff 0.0001777023
1226: 89a90128883ffff 0.0002372291
1227: 89a90166da7ffff 0.0001957669
```
:::
:::


As funções apresentadas nesta seção podem receber também outros *inputs* não explicitamente mencionados aqui. Para mais informações sobre cada um dos parâmetros, por favor consulte a documentação do pacote `accessibility` no seu [site](https://ipeagit.github.io/accessibility/).

## Cálculo de acessibilidade com o r5r

Ao longo das duas seções anteriores, nós mostramos como calcular níveis de acessibilidade passo-a-passo. Para fins didáticos, é importante entender que o cálculo de estimativas de acessibilidade tem como primeiro passo a geração de uma matriz de custos de viagens a partir de simulações de roteamento, que posteriormente é utilizada para estimar níveis de acessibilidade.

No entanto, o `r5r` inclui também uma função chamada `accessibility()` que faz todo esse processamento com uma única chamada. De forma parecida com a função de cálculo de matriz de tempo de viagem, a função de acessibilidade também recebe como *inputs* uma conexão com o R5, as origens, os destinos, os modos de transporte, o tempo de partida, entre outros. Adicionalmente, devem ser listadas também quais oportunidades devem ser consideradas e a função de decaimento que deve ser utilizada (bem como o valor do limite de custo e de parâmetro de decaimento). O exemplo abaixo mostra uma aplicação desta função.


::: {.cell}

```{.r .cell-code}
r5r_access <- accessibility(
  r5r_core,
  origins = points,
  destinations = points,
  opportunities_colname = "schools",
  decay_function = "step",
  cutoffs = 30,
  mode = c("WALK", "TRANSIT"),
  departure_datetime = as.POSIXct(
    "13-05-2019 14:00:00",
    format = "%d-%m-%Y %H:%M:%S"
  ),
  max_walk_dist = 800,
  max_trip_duration = 120,
  verbose = FALSE,
  progress = FALSE
)

r5r_access
```

::: {.cell-output .cell-output-stdout}
```
              from_id percentile cutoff accessibility
   1: 89a901291abffff         50     30            17
   2: 89a9012a3cfffff         50     30             0
   3: 89a901295b7ffff         50     30            13
   4: 89a901284a3ffff         50     30             0
   5: 89a9012809bffff         50     30            17
  ---                                                
1223: 89a90129133ffff         50     30             1
1224: 89a9012ac43ffff         50     30             8
1225: 89a90129a47ffff         50     30            10
1226: 89a90128883ffff         50     30            30
1227: 89a90166da7ffff         50     30            14
```
:::
:::


Como podemos observar, o resultado desta função são os níveis de acessibilidade já calculados. A informação “intermediária” de tempo de viagem entre origens e destinos, consequentemente, não fica disponível ao usuário com o uso da função. Ainda assim, este fluxo de trabalho pode ser uma boa alternativa para pessoas que estejam interessados puramente nos níveis de acessibilidade e não dependam do tempo de viagem em suas análises.

## Análises de acessibilidade

Calculados os níveis de acessibilidade, procedemos então para sua análise. Existe uma grande variedade de análises que podem ser feitas usando esses dados, por exemplo de diagnóstico das condições de acessibilidade urbana de diferentes bairros, pesquisas sobre desigualdades de acesso a oportunidades entre diferentes grupos sociais, análises sobre exclusão social e *accessibility poverty* (insuficiência de acessibilidade, tradução livre), etc. Nesta seção, no entanto,nos restringiremos a apresentar duas análises relativamente simples e de fácil comunicação: a distribuição espacial da acessibilidade e sua distribuição entre diferentes grupos de renda.

**Distribuição espacial de acessibilidade urbana**

Para compreendermos a distribuição espacial da acessibilidade urbana de uma determinada cidade ou região, primeiro precisamos obter as informações espaciais dos pontos que foram utilizados como origens e destinos no cálculo da matriz. Os pontos que nós usamos nos exemplos anteriores, por exemplo, correspondem a células de uma grade hexagonal baseadas no índice H3, desenvolvido pela Uber [@brodsky2018h3]. O código e o mapa a seguir mostram a distribuição desses hexágonos na nossa área de estudo.


::: {.cell}

```{.r .cell-code}
poa_grid <- read_grid("Porto Alegre")

poa_grid <- poa_grid[poa_grid$id_hex %in% points$id, ]

ggplot(poa_grid) + geom_sf()
```

::: {.cell-output-display}
![](3_calculando_acesso_files/figure-html/unnamed-chunk-14-1.png){width=672}
:::
:::


Para visualizarmos os nossos dados de acessibilidade espacialmente, portanto, nós precisamos unir a nossa tabela com estimativas de acessibilidade com a tabela que contém os dados espaciais da grade usando a coluna de `id` dos hexágonos como nossa coluna-chave. Os comandos usados para isso, bem como o resultado da operação em formato de imagem, podem ser vistos no código e no mapa a seguir.


::: {.cell}

```{.r .cell-code}
setDT(poa_grid)
poa_grid[cumulative_access, on = c(id_hex = "id"), schools_access := i.schools]

poa_grid_sf <- st_sf(poa_grid)

ggplot(poa_grid_sf) +
  geom_sf(aes(fill = schools_access), color = NA) +
  scale_fill_viridis_c(option = "inferno")
```

::: {.cell-output-display}
![](3_calculando_acesso_files/figure-html/unnamed-chunk-15-1.png){width=672}
:::
:::


Como podemos ver, os níveis de acessibilidade tendem a se concentrar de forma mais acentuada no centro da cidade, onde existe maior concentração de empregos, e próximos aos grandes corredores de transportes da cidade. Pessoas que moram mais perto desses corredores tendem a gastar menos tempo no acesso a suas estações e conseguem acessar locais distantes relativamente rápido, por terem fácil acesso a modos de alta capacidade e velocidade. Pessoas que moram mais longe desses corredores, por sua vez, dependem de modos de menor velocidade operacional (como os ônibus municipais, por exemplo) e precisam gastar mais tempo para alcançar os corredores de média e alta capacidade. Como consequência, seus níveis de acessibilidade tendem a ser menores.

**Distribuição socioeconômica de acessibilidade urbana**

O mapa acima, embora revelador quanto aos locais em que estão dispostas as maiores concentrações de acessibilidade, nada diz sobre quais são os grupos socioeconômicos que possuem os maiores potenciais de acesso a oportunidades na região. Para isso, nós precisamos cruzar informações demográficas e econômicas de cada um dos nossos pontos de origem com os dados de acessibilidade previamente calculados. No exemplo abaixo, nós juntamos aos dados de acessibilidade a informação do decil de renda de cada uma das origens. Assim, nós conseguimos identificar se um hexágono é de baixa, média ou alta renda. 

Os dados com as características socioeconômicas e perfil de renda que utilizamos no exemplo vêm do censo demográfico brasileiro de 2010, e foram agregados na grade espacial de dos hexágonos por @pereira2022usosolo. Aqui, nós acessamos os dados diretamente de dentro do R usando o pacote `aopdata`, que será apresentado em mais detalhes no capítulo Xx.


::: {.cell}

```{.r .cell-code}
poa_population <- read_population("Porto Alegre", showProgress = FALSE)

poa_grid[
  poa_population,
  on = "id_hex",
  `:=`(
    pop_count = i.P001,
    income_decile = i.R003
  )
]

poa_grid[, .(id_hex, schools_access, pop_count, income_decile)]
```

::: {.cell-output .cell-output-stdout}
```
               id_hex schools_access pop_count income_decile
   1: 89a9012124fffff             14       733             9
   2: 89a9012126bffff             13       355             9
   3: 89a9012127bffff             14       996            10
   4: 89a90128003ffff             34      1742             4
   5: 89a90128007ffff             15       477             5
  ---                                                       
1218: 89a90e934cbffff              7       118             4
1219: 89a90e934cfffff             12       518             6
1220: 89a90e934d3ffff              5       846             6
1221: 89a90e934d7ffff             12      1615             7
1222: 89a90e934dbffff              6         0            NA
```
:::
:::


Como vocês podem perceber, nós também trouxemos a informação de contagem populacional em cada origem. Isto se dá porque nós iremos calcular a distribuição da acessibilidade de cada um dos decis de renda. Para isso, portanto, nós precisamos ponderar o nível de acessibilidade de cada origem pela quantidade de pessoas que residem ali. Desta forma, nós teremos a distribuição da acessibilidade das pessoas localizadas em origens de um determinado decil de renda. Caso optássemos por não ponderar, no entanto, nós teríamos a distribuição de acessibilidade não das pessoas localizadas em cada hexágono, mas dos hexágonos em si. Como em nossa análise nós nos importamos com as pessoas, e não com as células espaciais, precisamos fazer a ponderação. Nós podemos visualizar a distribuição de acessibilidade de cada decil usando um boxplot, como feito a seguir.


::: {.cell}

```{.r .cell-code}
poa_grid[, income_decile := as.factor(income_decile)]

ggplot(poa_grid[!is.na(income_decile)]) +
  geom_boxplot(
    aes(
      x = income_decile,
      y = schools_access,
      color = income_decile,
      weight = pop_count
    )
  )
```

::: {.cell-output-display}
![](3_calculando_acesso_files/figure-html/unnamed-chunk-17-1.png){width=672}
:::
:::


O gráfico é muito claro em seu conteúdo: pessoas de mais baixa renda tendem a ter níveis de acessibilidade muito menores do que as de alta renda. Isso é um padrão comum em praticamente todas as cidades brasileiras e em diversas cidades do mundo [@pereira2019desigualdades]. Isto ocorre, em larga medida, devido à localização espacial das comunidades de baixa e alta renda no território: os mais ricos costumam morar em áreas mais valorizadas, próximas das grandes concentração de empregos (e oportunidades de educação, saúde, lazer, etc., também) e com maior oferta de transporte público de média e alta capacidade. Os mais pobres, por sua vez, tendem a morar em locais mais afastados, onde o valor da terra é menor. Consequentemente, tendem também a se afastar das grandes concentrações de oportunidades. Junta-se a isso o fato de, na maior parte dos casos, a oferta de serviços de transporte público de média e alta capacidade ser menor em locais com maior concentração de pessoas de baixa renda. Como consequência, seus níveis de acessibilidade são, em média, muito menores do que os dos mais ricos, como o gráfico deixa claro.