<div align="center">
<img src="https://static.wixstatic.com/media/efe4c3_128d08de3ab94815b7f619719ea5d21c~mv2.png/v1/fill/w_200,h_96,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/Dataside%20logo.png">
</div>


<h1 align="center">Dataside Challenge</h1>

---

## Objetivo
<p align="justify">O desafio que a Dataside propôs, foi o desenvolvimento de um notebook que será responsável por extrair 
dados de previsão do tempo das cidades do Vale do Paraíba, região onde se localiza a Dataside.</p>

## Passos

1. Consultar todas as cidades dessa região utilizando uma API para localização das cidades:

```Python
# Buscar cidades do Vale do Paraíba
request = requests.get("https://servicodados.ibge.gov.br/api/v1/localidades/mesorregioes/3513/municipios")
cities_info_dict = request.json()
```

2. Gerar um dataframe das cidade consultadas com os campos ID da cidade, nome da cidade e região:

```Python
# Criar data frame com as cidades
row_list = []
for city in cities_info_dict:
    row = {'ID_CITY': city['id'], 'CITY_NAME': city['nome'], 'CITY_REGION': city['microrregiao']['mesorregiao']['UF']['regiao']['nome']}
    row_list.append(row)
    
df_cities = spark.createDataFrame(row_list, schema='ID_CITY int, CITY_NAME string, CITY_REGION string')
```

3. Criar uma view temporária do dataframe gerado com as informações das cidades:

```Python
# Criar view com as cidades
df_cities.createOrReplaceTempView('CITIES')
```

4. Consultar dados geográficos (latitude e longitude) de cada cidade a partir da view das cidades utilizando uma API com essa funcionalidade:

```Python
# Buscar previsão do tempo para as cidades
translate_map = str.maketrans({'é': 'e', 'í': 'i', 'á': 'a', 'ç': 'c', 'ã': 'a', 'ô': 'o'})
row_list = []
for city in df_cities.collect():
    # API para buscar as coordenadas geográficas (latitude e longitude)
    geographic_coordinates_API = f"https://nominatim.openstreetmap.org/search?city={city['CITY_NAME']}&state=SP&country=BR&format=json"

    # Solicita os dados geográficos
    request = requests.get(geographic_coordinates_API)
    request_dict = request.json()

    latitude = request_dict[0]['lat']
    longitude = request_dict[0]['lon']
```

5. Com as coordenadas geográficas, consultar a previsão do tempo de cada cidade para os próximos 5 dias:

````Python
    # API para previsão do tempo
    forecast_weather_API = f"http://api.weatherapi.com/v1/forecast.json?key=c499600de8e14cb6804160909230701&q={latitude},{longitude}&days={days}&aqi=no&alerts=no&lang=pt"

    # Solicita dados de previsão do tempo da cidade
    request = requests.get(forecast_weather_API)
    forecast_request_dict = request.json()
    
    for d in range(0, days):
        forecast_day_dict = forecast_request_dict['forecast']['forecastday'][d]
        location_dict = forecast_request_dict['location']
        
        # Formatação de datas e horários
        date = datetime.fromisoformat(forecast_day_dict['date']).date()
        sunrise_time = datetime.strptime((forecast_day_dict['date'] + forecast_day_dict['astro']['sunrise']), '%Y-%m-%d%I:%M %p')
        sunset_time = datetime.strptime((forecast_day_dict['date'] + forecast_day_dict['astro']['sunset']), '%Y-%m-%d%I:%M %p')

        # Linha do registro de previsão do tempo
        row = {'Cidade': city['CITY_NAME'].translate(translate_map),
               'CodigoDaCidade': city['ID_CITY'],
               'Regiao': city['CITY_REGION'],
               'Data': date,
               'Pais': location_dict['country'],
               'Latitude': float(latitude),
               'Longitude': float(longitude),
               'TemperaturaMaxima': round(forecast_day_dict['day']['maxtemp_c']),
               'TemperaturaMinima': round(forecast_day_dict['day']['mintemp_c']),
               'TemperaturaMedia': round(forecast_day_dict['day']['avgtemp_c']),
               'VaiChover': forecast_day_dict['day']['daily_will_it_rain'],
               'ChanceDeChuva': forecast_day_dict['day']['daily_chance_of_rain'],
               'CondicaoDoTempo': forecast_day_dict['day']['condition']['text'],
               'NascerDoSol': sunrise_time,
               'PorDoSol': sunset_time,
               'VelocidadeMaximaDoVento': forecast_day_dict['day']['maxwind_mph']}

        row_list.append(row)
````

6. Gerar um dataframe das previsões consultadas com os campos ID da cidade, nome, região, data consultada, país, latitude, longitude, temperatua máxima, temperatura mínima, temperatura média, vai chover, chance de chuva, condição do tempo, nascer do sol, pôr do sol e velocidade máxima do vento:

```Python
# Criar data frame com as previsões
df_forecast = spark.createDataFrame(row_list, schema='Cidade string, CodigoDaCidade int, Regiao string,\
                                                      Data date, Pais string, Latitude float, Longitude float,\
                                                      TemperaturaMaxima int, TemperaturaMinima int, TemperaturaMedia int,\
                                                      VaiChover int, ChanceDeChuva int, CondicaoDoTempo string,\
                                                      NascerDoSol timestamp, PorDoSol timestamp, VelocidadeMaximaDoVento float')
```

7. Criar uma view temporária do dataframe gerado com as previsões:

```Python
# Criar view com as previsões
df_forecast.createOrReplaceTempView('FORECASTS')
```

8. Gerar um dataframe da Tabela 1 e 2, a partir da view das previsões:

```Python
# Criar DF da Tabela 1
df_table1 = spark.sql(f'SELECT Cidade, CodigoDaCidade, Regiao, Data,\
                       Pais, Latitude, Longitude, TemperaturaMaxima,\
                       TemperaturaMinima, TemperaturaMedia,\
                       case VaiChover WHEN 1 THEN "Sim" WHEN 0 THEN "Não" END VaiChover,\
                       ChanceDeChuva,\
                       CondicaoDoTempo,\
                       date_format(NascerDoSol, "HH:mm:ss") NascerDoSol,\
                       date_format(PorDoSol, "HH:mm:ss") PorDoSol,\
                       VelocidadeMaximaDoVento\
                       FROM FORECASTS\
                       WHERE Data BETWEEN "{initial_date}" AND "{end_date}"')

# Criar DF da Tabela 2
df_table2 = spark.sql(f'SELECT Cidade, \
                       COUNT(CASE VaiChover WHEN 1 THEN 1 END) AS QtdDiasVaiChover, \
                       COUNT(CASE VaiChover WHEN 0 THEN 1 END) AS QtdDiasNaoVaiChover, \
                       COUNT(Data) AS TotalDiasMapeados \
                       FROM FORECASTS \
                       WHERE Data BETWEEN "{initial_date}" AND "{end_date}" \
                       GROUP BY Cidade')

```

9. Exportar o data frame para formato CSV:

```Python
# Exportar CSVs
df_table1.write.option('header', True).csv(r"C:\Tabela1", mode='overwrite')
df_table2.write.option('header', True).csv(r"C:\Tabela2", mode='overwrite')
```

Obs.: Para definifição das datas e dias a serem consultados foi utilizado o código abaixo:
```Python
# Seleciona a data incial (dia atual) e data final (5 dias após o dia atual)
initial_date = datetime.today().date()
end_date  = initial_date + timedelta(days=6)
days = abs(initial_date - end_date).days
```

Dessa forma, seria possível automatizar o software para sempre consultar os proximos 5 dias ou ainda controlar a saída de dados buscando por dias específicos.

## Tecnologias usadas
- API's:
  -  [Documentação API IBGE — Localidades](https://servicodados.ibge.gov.br/api/docs/localidades)
  - [Documentação API Dados Geográficos]()




