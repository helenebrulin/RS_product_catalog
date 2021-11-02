
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

>HSET product:1 ... # table
>HSET product:2 ... #tablecloth, flower pattern in description
>HSET product:3 ... # computer
>HSET product:4 ... # glasses : description about table
>HMGET ...


>HSET vendor:1 ...
>HSET vendor:2 ...

## Create indexes

> FT.CREATE idx:product ON hash PREFIX 1 "product:" SCHEMA name TEXT SORTABLE release_year NUMERIC SORTABLE rating NUMERIC SORTABLE genre TAG SORTABLE
> FT.CREATE idx:vendor ON hash PREFIX 1 "vendor:" SCHEMA name TEXT SORTABLE release_year NUMERIC SORTABLE rating NUMERIC SORTABLE genre TAG SORTABLE

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
-> Tablecloth is returned = This is because the name has been indexed as text, so the field is tokenized and stemmed.


It is also possible to limit the list of fields returned by the query using the RETURN parameter:
>FT.SEARCH idx:product "table" RETURN 2 name vendor


This query does not specify any "field" and still returns some products, this is because RediSearch will search all TEXT fields by default. In the current index only the title is present as a TEXT field. You will see later how to update an index, to add more fields to it.

If you need to perform a query on a specific field you can specify it using the @field: syntax, for example:
>FT.SEARCH idx:product "@name:table" RETURN 2 name vendor

All the products containing the string "table" but not the flower one :
>FT.SEARCH idx:product "table -flower" RETURN 2 name vendor

Fuzzysearch : 
>FT.SEARCH idx:product "%compter%" RETURN 2 name vendor

## Search tags

Category:
The category field is indexed as a TAG and allows exact match queries 
>FT.SEARCH idx:product "@category:{furniture|linen}" RETURN 2 name vendor

Combine:
>FT.SEARCH idx:product "@category:{furniture|linen} @brand:-BRAND" RETURN 2 name vendor


## Search numerical range
>FT.SEARCH idx:product "@price:[50 200]" RETURN 3 name vendor price
>FT.SEARCH idx:product "@price:[50 (200]" RETURN 3 name vendor price # exclude a value


## Optional search
- show scores with optional conditions on tags : delivery + show TIDF + weights
recommendation : https://redislabs.moodlecloud.com/mod/page/view.php?id=985 

## Insert, Update, Delete and Expire
Let's count our index : FT.SEARCH idx:product "*" LIMIT 0 0
HSET a new one and count again. 
THIS NEW ONE SHOULD HAVE 0 in QUANTITY.

Now do a quantity search.
>HINCRBY product quantity 2 # show indexing immediate after change
Do quantity search again. 

When you delete the hash, the index is also updated, and the same happens when the key expires (TTL-Time To Live).
For example, set the product  to expire in 20 seconds time:
> EXPIRE product:x 20

SEARCH FOR IT AFTER 20 SECONDS. 

> FT.SEARCH "idx:movie" "@genre:{Action}" LIMIT 0 0 count based on quantity

-> discuss ephemeral search : https://redislabs.com/blog/the-case-for-ephemeral-search/ 


## Sort
Sort based on price : 
> FT.SEARCH "idx:movie" "@genre:{Action}"  SORTBY release_year DESC RETURN 2 title release_year
> Note: The field used in the SORTBY should be part of the index schema and defined as SORTABLE.


## Geo-search on vendors
> SEARCH within 40m radius : FT.SEARCH "idx:theater" "@location:[-73.9798156 40.7614367 400 m]" RETURN 2 name address
> Find the closest : AGGREGATION

## Other interesting aggregations
- nb of product by vendor
> FT.AGGREGATE "idx:movie" "*" GROUPBY 1 @release_year REDUCE COUNT 0 AS nb_of_movies

- nb of product by vendor from the biggest to lowest
> FT.AGGREGATE "idx:movie" "*" GROUPBY 1 @release_year REDUCE COUNT 0 AS nb_of_movies SORTBY 2 @release_year DESC 

- SAME AS ABOVE with a condition of quantity
> FT.AGGREGATE idx:user "@gender:{female}" GROUPBY 1 @country  REDUCE COUNT 0 AS nb_of_users  FILTER "@country!='china' && @nb_of_users > 100" SORTBY 2 @nb_of_users DESC


- nb of product by category with average price
FT.AGGREGATE idx:movie "*" GROUPBY 1 @genre REDUCE COUNT 0 AS nb_of_movies REDUCE SUM 1 votes AS nb_of_votes REDUCE AVG 1 rating AS avg_rating SORTBY 4 @avg_rating DESC @nb_of_votes DESC


## OTHER
- show stem search, autocomplete

