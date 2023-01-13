<div align="center">
<img src="https://static.wixstatic.com/media/efe4c3_128d08de3ab94815b7f619719ea5d21c~mv2.png/v1/fill/w_200,h_96,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/Dataside%20logo.png">
</div>


<h1 align="center">Dataside Challenge</h1>

---

## Objetivo
<p align="justify">O desafio que a Dataside propôs, foi o desenvolvimento de um notebook que será responsável por extrair 
dados de previsão do tempo das cidades do Vale do Paraíba, região onde se localiza a Dataside.</p>

## Passos
```
1 - Consultar todas as cidades dessa região utilizando uma API para localização das cidades;
2 - Gerar um dataframe das cidade consultadas com os campos ID da cidade, nome da cidade e região;
3 - Criar uma view temporária do dataframe gerado com as informações das cidades;
4 - Consultar dados geográficos (latitude e longitude) de cada cidade a partir da view das cidades utilizando uma API com essa funcionalidade;
5 - Com as coordenadas geográficas, consultar a previsão do tempo de cada cidade para os próximos 5 dias;
6 - Gerar um dataframe das previsões consultadas com os campos ID da cidade, nome, região, data consultada, país, latitude, longitude, temperatua máxima, temperatura mínima, temperatura média, vai chover, chance de chuva, condição do tempo, nascer do sol, pôr do sol e velocidade máxima do vento;
7 - Criar uma view temporária do dataframe gerado com as previsões;
8 - Gerar um dataframe da Tabela 1 e 2, a partir da view das previsões;
9 - Exportar o data frame para CSV;
```


[Documentação API IBGE — Localidades](https://servicodados.ibge.gov.br/api/docs/localidades)




