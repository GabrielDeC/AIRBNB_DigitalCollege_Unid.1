# Análise de Dados Airbnb do Rio de Janeiro

Este repositório tem como objetivo aplicar os conhecimentos adquiridos na Unidade 1 do Curso de Data Analytics da Digital College. Desenvolvi um banco de dados relacional para armazenar e analisar dados de mercado do Airbnb do Rio de Janeiro e, com isso, extrai insights sobre ocupação, precificação e oportunidades de investimento.

# Passo-a-Passo da Construção do BRModelo e do Data Warehouse

O caminho inicial para a confecção do DataWarehouse primeiro foi a consulta à fonte de dados que iria ser utilizada neste caso o site [Inside Airbnb](https://insideairbnb.com/get-the-data/) que traz informações interessantes sobre as locações de Airbnb em diversas cidades e países. Iremos focar nos dados fornecidos para o Rio de Janeiro que teve como data da última atualização durante a análise dos dados aqui presentes 26/Setembro/2025.

Diante dos dados "crus" tive que analisar o que era fornecido em cada uma dos CSVs disponibilizados: *listings*, *neighbourhood* e *reviews*. Além disso, no site existe a disponibilização do [Dicionário Inside Airbnb](https://docs.google.com/spreadsheets/d/1iWCNJcSutYqpULSQHlNyGInUvHg2BoUGoNRIGa6Szc4/edit?gid=1322284596#gid=1322284596) para entender melhor o que cada coluna fornece de informação.

Diante disso, acessando o [BR Modelo](https://app.brmodeloweb.com/) criei o modelo conceitual abaixo respeitando a estrutura base dos CSVs fornecidos no site.

<img width="701" height="495" alt="image" src="https://github.com/user-attachments/assets/f3086b79-6b16-44a7-ba43-33af92885c60" />

A mudança que fiz foi adicionar um ID Serial para a entidade *neighbourhood* e *reviews* para a criação de *foreign keys* mais adiante e definindo o relacionamento 1:N entre *listings* - *neighbourhood* e *listings* - *reviews*.

Com o modelo conceitual criado, foi gerado o seguinte modelo lógico.

<img width="362" height="476" alt="image" src="https://github.com/user-attachments/assets/f7e5e572-6559-4f05-b1be-55b73b43d909" />

Utilizei as queries geradas pelo próprio BRModelo, no entanto, devido a algumas limitações adicionei e ajustei algumas queries para melhor modelagem, conforme definido abaixo.

#### Query para Tabela *reviews*
```sql
CREATE TABLE reviews 
( 
 review_id SERIAL PRIMARY KEY,
 listing_id BIGINT,
 data DATE     
)
```

#### Query para Tabela *neighbourhood*
```sql
CREATE TABLE neighbourhood ( 
    id_nb SERIAL PRIMARY KEY,
    neighbourhood_group VARCHAR,  
    neighbourhood VARCHAR  
)
```

#### Query para Tabela *listings*
```sql
CREATE TABLE listings (
    id BIGINT PRIMARY KEY,
    name TEXT,
    host_id BIGINT,
    host_name VARCHAR(255),
    neighbourhood_group VARCHAR(255),
    neighbourhood VARCHAR(255),
    latitude FLOAT,
    longitude FLOAT,
    room_type VARCHAR(100),
    price FLOAT,
    minimum_nights INT,
    number_of_reviews INT,
    last_review DATE,
    reviews_per_month FLOAT,
    calculated_host_listings_count INT,
    availability_365 INT,
    number_of_reviews_ltm INT,
    license TEXT
)
```

#### Query para Relacionamento 1:N entre *listings* e *reviews*
```sql
ALTER TABLE reviews 
ADD CONSTRAINT fk_listings_reviews 
FOREIGN KEY (listing_id) REFERENCES listings (id)
```

#### Query para adicionar *foreign key* de *neighbourhood* à *listings*
```sql
ALTER TABLE listings ADD COLUMN id_nb_fk INT
```

#### Query para Relacionamento entre *listings* e *neighbourhood*
```sql
UPDATE listings l
SET id_nb_fk = n.id_nb
FROM neighbourhood n
WHERE l.neighbourhood = n.neighbourhood
```

#### Query para adicionar *foreign key* de *listings* à *neighbourhood*
```sql
ALTER TABLE listings 
ADD CONSTRAINT fk_listings_neighbourhood 
FOREIGN KEY (id_nb_fk) REFERENCES neighbourhood (id_nb)
```

---

# Problemas de Negócios - Airbnb Rio de Janeiro

Este projeto utiliza SQL para extrair análises importantes de uma base de dados com mais de 43 mil anúncios de Airbnb na cidade do Rio de Janeiro. 
Abaixo estão detalhados os 10 principais indicadores desenvolvidos para apoiar a tomada de decisão.

---

## Análises de Mercado e Performance

### 1. Ticket Médio por Bairro
O objetivo principal desse indicador é identificar o valor médio cobrado por noite em cada região. Sendo essencial para entender o posicionamento de preços e o perfil socioeconômico de cada bairro.

```sql
SELECT 
    n.neighbourhood AS bairro,
    AVG(l.price)::numeric AS ticket_medio
FROM 
    listings l
JOIN 
    neighbourhood n ON l.id_nb_fk = n.id_nb
GROUP BY 
    n.neighbourhood, 
    n.neighbourhood_group
ORDER BY 
    ticket_medio DESC;
```

### 2. Taxa de Ocupação Percentual
Neste cálculo temos a estimativa de dias ocupados com base na disponibilidade anual (`availability_365`). 
Indica a eficiência de reserva de cada região.

```sql
SELECT 
    n.neighbourhood AS bairro,
    ROUND(AVG(l.availability_365), 1) AS media_dias_disponiveis,
    ROUND(
        AVG((365 - l.availability_365) / 365.0) * 100, 2
    ) AS taxa_ocupacao_percentual
FROM 
    listings l
JOIN 
    neighbourhood n ON l.id_nb_fk = n.id_nb
GROUP BY 
    n.neighbourhood
ORDER BY 
    taxa_ocupacao_percentual DESC;
```

### 3. Bairros com Maior Número de Airbnbs
Com o objetivo de medir a quantidade de ofertas por bairro, podemos utilizar a query abaixo e ajuda a identificar os locais mais procurados e consolidados e a concentração de mercado da região.

```sql
SELECT 
    n.neighbourhood AS bairro,
    COUNT(l.id) AS total_anuncios
FROM 
    listings l
JOIN 
    neighbourhood n ON l.id_nb_fk = n.id_nb
GROUP BY 
    n.neighbourhood 
ORDER BY 
    total_anuncios DESC;
```

### 4. Análise de Oportunidade e Saturação
Aqui cruzei os dados de volume de oferta e disponibilidade média para ordenar quais bairros têm mais oportunidade diante da quantidade de anúncios.

```sql
SELECT 
    neighbourhood AS bairro,
    COUNT(*) AS total_imoveis,
    ROUND(AVG(price)::numeric, 2) AS preco_medio,
    ROUND(AVG(availability_365)::numeric, 2) AS disponibilidade_media,
    SUM(number_of_reviews) AS total_reviews_acumuladas,
    CASE 
        WHEN AVG(availability_365) < 150 AND COUNT(*) > 1000 THEN 'Alta Demanda / Saturado'
        WHEN AVG(availability_365) > 250 AND COUNT(*) > 500 THEN 'Possível Excesso de Oferta'
        WHEN AVG(availability_365) < 150 AND COUNT(*) < 200 THEN 'Oportunidade: Pouca Oferta / Alta Procura'
        ELSE 'Mercado em Equilíbrio'
    END AS status_mercado
FROM listings
GROUP BY neighbourhood
HAVING COUNT(*) > 5
ORDER BY total_imoveis DESC;
```

---

## Análises de Popularidade e Reputação

### 5. Bairros Mais Avaliados (Total de Reviews)
O volume total de comentários serve como um indicador para o volume de transações reais ocorridas em cada região.

```sql
SELECT 
    l.neighbourhood AS bairro,
    SUM(l.number_of_reviews)::numeric AS reviews_total
FROM 
    listings l
GROUP BY 
    l.neighbourhood
ORDER BY 
    reviews_total DESC;
```

### 6. Hosts com Mais Avaliações
O volume de avaliações permite dentificar os principais anfitriões da plataforma, permitindo distinguir entre usuários comuns e pessoas que utilizam o airbnb profisionalmente.

```sql
SELECT 
    l.host_name AS nome_host,
    SUM(l.number_of_reviews)::numeric AS reviews_total
FROM 
    listings l
GROUP BY 
    l.host_name
ORDER BY 
    reviews_total DESC;
```

### 7. Média de Avaliações Mensais por Host (Giro Mensal)
Mede a velocidade com que um anfitrião recebe novas avaliações, servindo como indicador de rotatividade operacional.

```sql
SELECT 
    l.host_name AS nome_host,
    ROUND(AVG(l.reviews_per_month)::numeric, 2) AS giro_mensal_medio
FROM 
    listings l
WHERE 
    l.reviews_per_month IS NOT NULL
GROUP BY 
    l.host_name
ORDER BY 
    giro_mensal_medio DESC;
```

### 8. Média de Avaliação LTM (Last Twelve Months) por Host
Foca na performance recente (últimos 12 meses), eliminando anúncios muito antigos que podem não refletir a qualidade atual.

```sql
SELECT 
    l.host_name AS nome_host,
    ROUND(AVG(l.number_of_reviews_ltm), 2) AS media_reviews_ultimo_ano
FROM 
    listings l
GROUP BY 
    l.host_name
ORDER BY 
    media_reviews_ultimo_ano DESC;
```

---

## Rankings de Listings (Destaques Individuais)

### 9. 10 Maiores Airbnbs em Número de Avaliações
Anúncios que possuem o maior histórico de sucesso e validação social na cidade.

```sql
SELECT 
    id,
    name AS anuncio,
    host_name AS anfitriao,
    neighbourhood AS bairro,
    number_of_reviews AS total_avaliacoes,
    price AS preco_noite
FROM listings
ORDER BY number_of_reviews DESC
LIMIT 10;
```

### 10. 10 Airbnbs Mais Caros
Mapeia o segmento de luxo do Rio de Janeiro, identificando os anúncios com os maiores valores por noite.

```sql
SELECT 
    id,
    name AS anuncio,
    host_name AS anfitriao,
    neighbourhood AS bairro,
    number_of_reviews AS total_avaliacoes,
    price AS preco_noite
FROM listings l
WHERE 
    l.price IS NOT NULL
ORDER BY preco_noite DESC
LIMIT 10;
```
