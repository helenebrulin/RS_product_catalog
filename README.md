
# RediSearch 2.0 Product Catalog demo

docker run -p 6379:6379 redislabs/redisearch:latest

## 1 - Setup RE database with RedisSearch

## 2 - Insert some product data

A product is represented by the following attributes : 
- id 
- name
- description 
- category
- brand 
- vendor_id
- delivery_tags : collect - home_delivery
- price : numerical
- quantity : 

Let's do vendor hashes with coordinates as well :
- vendor_id
- name  
- location


>HSET product:1 name "Kitchen Table" description "Small wooden table for all your appliances." category "Furniture" brand "Ikea" vendor_id 1 delivery "Home" price 50 quantity 1 comments 160

>HSET product:2 name "Telescope" description "To infinity and beyond" category "Astronomy" brand "Sky" vendor_id 2 delivery "Home" price 200 quantity 0 comments 855

>HSET product:3 name "Plates" description "Sold by 6. For beautiful tables" category "Kitchenware" brand "Ceramica" vendor_id 2 delivery "Collect" price 165 quantity 5 comments 4

>HSET product:4 name "Macbook Pro" description "Our best selling computer" category "Computers" brand "Apple" vendor_id 3 delivery "Home" price 1000 quantity 100 comments 999

HSET product:5 name "HP Computer" description "I'm a windows computer." category "Computers" brand "HP Computers" vendor_id 3 delivery "Collect" price 300 quantity 50 comments 753

>HSET vendor:1 name "Ikea" location "-66.94198,-34.15366"
>HSET vendor:2 name "HomeDepot" location "103.79232,34.26015"
>HSET vendor:3 name "AppleStore Marina Bay" location "99.94961,11.88799"


## Create indexes

> FT.CREATE idx:product ON hash PREFIX 1 "product:" SCHEMA name TEXT SORTABLE description TEXT SORTABLE category TAG SORTABLE brand TEXT SORTABLE vendor_id NUMERIC SORTABLE delivery TAG SORTABLE price NUMERIC SORTABLE quantity NUMERIC SORTABLE comments NUMERIC SORTABLE

> FT.CREATE idx:vendor ON hash PREFIX 1 "vendor:" SCHEMA name TEXT SORTABLE location GEO SORTABLE

TEXT : allows full text search queries
TAG : allows exact match queries
GEO : allows geographic range queries
NUMERIC : allows numeric range queries 
SORTABLE : numeric, tag or text fields can have the optional SORTABLE argument that allows the user to later sort the results by the value of this field. 

> FT.INFO idx:product
> FT.INFO idx:vendor

## Search full-text

>FT.SEARCH idx:product "table"
-> returns all the items that contain "table" in all fields.
-> Plates is returned = This is because the name has been indexed as text, so the field is tokenized and stemmed.


It is also possible to limit the list of fields returned by the query using the RETURN parameter:
>FT.SEARCH idx:product "table" RETURN 2 name vendor_id


This query does not specify any "field" and still returns some products, this is because RediSearch will search all TEXT fields by default. In the current index only the title is present as a TEXT field. You will see later how to update an index, to add more fields to it.

If you need to perform a query on a specific field you can specify it using the @field: syntax, for example:
>FT.SEARCH idx:product "@name:table" RETURN 2 name vendor_id

All the products containing the string "table" but not the "beautiful" one :
>FT.SEARCH idx:product "table -beautiful" RETURN 2 name vendor_id
OR
>>FT.SEARCH idx:product "table @description:-beautiful" RETURN 2 name vendor_id


Fuzzysearch : 
>FT.SEARCH idx:product "%compter%" RETURN 2 name vendor_id

## Search tags

Category:
The category field is indexed as a TAG and allows exact match queries 
>FT.SEARCH idx:product "@category:{furniture|Kitchenware}" RETURN 2 name vendor_id

Combine:
>FT.SEARCH idx:product "@category:{furniture|linen} @brand:-Ceramica" RETURN 2 name vendor_id


## Search numerical range
>FT.SEARCH idx:product "@price:[50 200]" RETURN 3 name vendor_id price
>FT.SEARCH idx:product "@price:[50 (200]" RETURN 3 name vendor_id price # exclude a value

## Sort
Sort based on price : 
> FT.SEARCH idx:product * SORTBY price DESC RETURN 2 name price
> Note: The field used in the SORTBY should be part of the index schema and defined as SORTABLE.


## Optional search
- show scores with optional conditions on tags : delivery + show TIDF + weights

>FT.SEARCH idx:product "~@delivery:{Collect}" RETURN 2 name delivery

Note that you can add several things with OR condition, but here it wouldn't make much sense.

>FT.SEARCH idx:product "~@delivery:{Collect}" RETURN 2 name delivery WITHSCORES


## Scoring with TFIDF
So let's look into how we could do recommendations based on scoring : We have two computers, right? SHow them and show that one has more times the word computer than the other. 
>FT.SEARCH idx:product "computer" WITHSCORES
We see that HP has highest score because the word computer appears more often.

>FT.SEARCH idx:product "computer" WITHSCORES EXPLAINSCORE

But we can weigh our conditions and make some of them more important : 
>FT.SEARCH idx:product "computer ~@brand:Apple => {$weight: 2.0}" WITHSCORES 
> add EXPLAINSCORE and see the weight added

You can also add those weights directly when you create your Index with FT.CREATE, so that you don't need to apply these weights in your searches.


## Geo-search on vendors
SEARCH within 400m radius : 
> FT.SEARCH "idx:vendor" "@location:[-66.94198 -34.15366 400 m]" RETURN 1 name

Now let's say we want to find the closest vendor to our user
> FT.AGGREGATE idx:vendor * LOAD 2 @name @location APPLY "geodistance(@location, 134.811767,51.6716705)" AS dist SORTBY 2 @dist ASC LIMIT 0 1


## Other interesting aggregations 
- nb of product by vendor
> FT.AGGREGATE "idx:product" "*" GROUPBY 1 @vendor_id REDUCE COUNT 0 AS nb_of_products

- nb of product by vendor and sort by nb
> FT.AGGREGATE "idx:product" "*" GROUPBY 1 @vendor_id REDUCE COUNT 0 AS nb_of_products SORTBY 2 @nb_of_products DESC 

- nb of product by category with total quantity and average price
FT.AGGREGATE idx:product "*" GROUPBY 1 @category REDUCE COUNT 0 AS nb_of_products REDUCE SUM 1 quantity AS total_quantity REDUCE AVG 1 price AS avg_price SORTBY 4 @avg_price DESC @total_quantity DESC

- Using filter
> FT.AGGREGATE idx:product * GROUPBY 1 @category REDUCE COUNT 0 AS nb_of_products FILTER "@category!='computers' && @nb_of_products >= 1" SORTBY 2 @nb_of_products DESC

- Now adding the notion of quantity 
> FT.AGGREGATE idx:product * GROUPBY 1 @category REDUCE COUNT 0 AS nb_of_products REDUCE SUM 1 quantity AS total_quantity FILTER "@category!='computers' && @total_quantity >= 1" SORTBY 2 @total_quantity DESC

HINCRBY product:2 quantity 2 and show telescope is now here, with quantity 2




!!! Add votes stuff : https://github.com/RediSearch/redisearch-getting-started/blob/master/docs/008-aggregation.md

## End
Open to slide with features and talk maybe about TTL and ephemeral search

