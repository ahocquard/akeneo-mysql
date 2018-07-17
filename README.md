# Akeneo Mysql

Goal of this repository is to use the new available features in Mysql 8.0 with Akeneo PIM.

## Requirement 

- [Docker Engine](https://docs.docker.com/engine/installation/)

## Installation

Available catalogs are "icecat_dump" or "eighty_percent_dump".

The first one is the default catalog installed by the PIM.
The second one is a biggest catalog, having 5000 categories, 125 families having 100 attributes, almost 1000 products with approximately 380 product values per products.

This catalog were generated in PIM 2.1.

```
$ CATALOG=icecat_dump
$ LOCAL_MYSQL_PORT=3306
$ docker run -d --name mysql-new-features -p ${LOCAL_MYSQL_PORT}:3306 -e MYSQL_ROOT_PASSWORD=root -e MYSQL_USER=akeneo_pim -e MYSQL_PASSWORD=akeneo_pim -e MYSQL_DATABASE=akeneo_pim mysql:8.0.11
$ wget -O - https://github.com/ahocquard/akeneo-mysql/raw/master/${CATALOG}.sql.gz | gunzip | docker exec -i mysql-new-features /usr/bin/mysql -u akeneo_pim --password=akeneo_pim akeneo_pim
```

Then, open the mysql client:
```
docker run -it --link mysql-new-features:mysql --rm mysql sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uakeneo_pim -p"akeneo_pim" akeneo_pim'
```

## Number of products values

Return the average number of product values for the 10000 first products.

```
SELECT AVG(length) FROM (
    SELECT JSON_LENGTH(JSON_EXTRACT(raw_values, '$.*.*.*')) AS length 
    FROM pim_catalog_product LIMIT 10000
) as product_length;
```

## Recursive queries

### Categories with path

```
WITH RECURSIVE category_tree AS
    (
        SELECT c.id, c.code, c.code as path
        FROM pim_catalog_category c
        WHERE c.parent_id IS NULL
        UNION ALL
        SELECT c.id, c.code, CONCAT(t.path, '/', c.code) as path
        FROM pim_catalog_category c JOIN category_tree t on c.parent_id = t.id
    )
SELECT * FROM category_tree;
```

|id  | code           | path                 |
|----|----------------|----------------------|
|1   |master	      |master                |
|2	 |sales	          |sales                 |
|160 |print	          |print                 |
|164 |suppliers	      |suppliers             |
|3	 |tvs_projectors  |master/tvs_projectors |
|6	 |cameras	      |master/cameras        |
|10	 |audio_video	  |master/audio_video    |


### Categories with parent categories

```
WITH RECURSIVE category_tree AS
(
    SELECT c.id, c.code, JSON_ARRAY() as parent_categories 
    FROM pim_catalog_category c
    WHERE c.parent_id IS NULL
    UNION ALL
    SELECT c.id, c.code, JSON_ARRAY_APPEND(t.parent_categories, '$', t.code) as path
    FROM pim_catalog_category c JOIN category_tree t on c.parent_id = t.id
)
SELECT * FROM category_tree;
```

|id  | code           | parent_categories |
|----|----------------|-------------------|
|1   |master	      |[]                 |
|2	 |sales	          |[]                 |
|160 |print	          |[]                 |
|164 |suppliers	      |[]                 |
|3	 |tvs_projectors  |["master"]         |
|6	 |cameras	      |["master"]         |
|10	 |audio_video	  |["master"]         |


### Variant products with inherited values from product models

```
WITH RECURSIVE complete_variant_products AS
(
    SELECT p.identifier, p.product_model_id as parent_id, p.raw_values
    FROM pim_catalog_product p
    WHERE p.product_model_id IS NOT NULL
    UNION ALL
    SELECT vp.identifier, pm.parent_id, JSON_MERGE_PATCH(pm.raw_values, vp.raw_values)
    FROM pim_catalog_product_model pm JOIN complete_variant_products vp on pm.id = vp.parent_id
)
SELECT identifier, raw_values FROM complete_variant_products WHERE parent_id IS NULL
```


## JSON Array Aggregation

### Categories with parent categories (without CTE)

```
SELECT path.node_code, JSON_ARRAYAGG(path.parent_code)
FROM (
    SELECT node.code as node_code, parent.code as parent_code
    FROM pim_catalog_category AS node,
    pim_catalog_category AS parent
    WHERE node.lft BETWEEN parent.lft AND parent.rgt
    ORDER BY parent.lft
) as path
GROUP BY path.node_code;
```

### Categories with translations

As translations is not mandatory, we have to use a `LEFT JOIN`.
The reason is that `ct.locale` can be `null`, preventing us to use `JSON_OBJECTAGG`, because it would mean that the key could be null, which is not allowed with JSON.

```
    SELECT 
        c.code,
        parent.code as parent_code,
        JSON_ARRAYAGG(
            JSON_OBJECT(
                'locale', ct.locale,
                'label', ct.label
            )
        ) as translations
    FROM pim_catalog_category c
    LEFT JOIN pim_catalog_category parent on parent.id = c.parent_id
    LEFT JOIN pim_catalog_category_translation ct on ct.foreign_key = c.id
    GROUP BY c.code
```

|code  |parent_code|translations                                                                                                            |
|------|-----------|------------------------------------------------------------------------------------------------------------------------|
|3go   |brands     |[{"label": "3GO", "locale": "en_US"}, {"label": "3GO", "locale": "de_DE"}, {"label": "3GO", "locale": "fr_FR"}]         |
|3m    |brands     |[{"label": "3M", "locale": "en_US"}, {"label": "3M", "locale": "de_DE"}, {"label": "3M", "locale": "fr_FR"}]            |
|a4tech|brands     |[{"label": "A4Tech", "locale": "en_US"}, {"label": "A4Tech", "locale": "de_DE"}, {"label": "A4Tech", "locale": "fr_FR"}]|

## JSON table

```
SELECT
	p.identifier,
    js.*
FROM
	pim_catalog_product p,
	JSON_TABLE (
		JSON_KEYS(p.raw_values),
        "$[*]" COLUMNS (
            rowid FOR ORDINALITY,
            col VARCHAR(50) PATH "$"
        )
	) as js
```

|identifier               |row_id| attribute|
|-------------------------|------|----------|
|Biker-jacket-polyester-xl|1     |ean       |
|Biker-jacket-polyester-xl|2     |sku       |
|Biker-jacket-polyester-xl|3     |size      |
|tvsam32                  |1     |sku       |
|tvsam32                  |2     |picture   |
|tvsam32                  |3     |size      |

