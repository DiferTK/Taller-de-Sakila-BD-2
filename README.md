# Taller-de-Sakila-BD-2
# Registro de Consultas SQL
En la presente, se mostraran diez consultas que se realizaron para la Base de Datos Sakila en MySQL

## 1. Insertar un registro en la tabla 'film'

Se inserta un registro en la tabla 'film' utilizando valores ficticios para asegurar la integridad referencial con otras tablas.

```sql
SELECT * FROM sakila.film;
INSERT INTO film (film_id,title, description, release_year, language_id, original_language_id, rental_duration, rental_rate, length, replacement_cost, rating, special_features, last_update)
VALUES (1001,'Adventuras del mar', 'Piratas pero de tematica romantica', 2002, 6, NULL, 10, 4.30, 113, 10.79, 'R', 'Trailers,Deleted Scenes,2007-03-14 05:30:42');
```
![Ejercicio 1](https://github.com/DiferTK/Taller-de-Sakila-BD-2/assets/154281253/296788e7-7dd7-4ef2-baae-a1897136675b)

##2. Películas con duración mayor que el promedio

Esta consulta selecciona las películas cuya duración es mayor que la duración promedio de todas las películas en la tabla 'film'.
```sql
USE sakila;
SELECT title, length
FROM film
WHERE length > (SELECT AVG(length) FROM film);
```
![Ejercicio 2](https://github.com/DiferTK/Taller-de-Sakila-BD-2/assets/154281253/f9bd7c15-1dfe-48ef-9a87-675f36298d20)

##3. ¿Qué películas están actualmente alquiladas en la tienda con store_id = 1?

```sql
SELECT 
    film.title 
FROM 
    film 
    INNER JOIN inventory ON film.film_id = inventory.film_id 
    INNER JOIN rental ON inventory.inventory_id = rental.inventory_id 
WHERE 
    rental.return_date IS NULL 
    AND inventory.store_id = 1;
```
![image](https://github.com/DiferTK/Taller-de-Sakila-BD-2/assets/154281253/723094b8-d127-4128-a95e-cfba7f3bcd80)

##4.De las películas en la tienda con store_id = 1, ¿cuáles fueron alquiladas por un período más largo que el período de alquiler promedio?

```sql
SELECT 
    film.title
FROM 
    film
    INNER JOIN inventory ON film.film_id = inventory.film_id
    INNER JOIN rental ON inventory.inventory_id = rental.inventory_id
WHERE 
    inventory.store_id = 1
GROUP BY 
    film.title
HAVING 
    AVG(TIMESTAMPDIFF(DAY, rental.rental_date, rental.return_date)) > (
        SELECT 
            AVG(TIMESTAMPDIFF(DAY, rental_date, return_date))
        FROM 
            rental
    );
```
![image](https://github.com/DiferTK/Taller-de-Sakila-BD-2/assets/154281253/d5e2f740-c447-4741-917f-de96ca629205)

##5.Qué actores forman parte del reparto de 5 o menos películas? (Revisar)
Esta consulta selecciona los actores que han participado en 5 o menos películas
```sql
SELECT 
    actor.actor_id,
    CONCAT(actor.first_name, ' ', actor.last_name) AS actor_name,
    COUNT(*) AS movie_count
FROM 
    actor
    INNER JOIN film_actor ON actor.actor_id = film_actor.actor_id
GROUP BY 
    actor.actor_id
HAVING 
    movie_count <= 5;
```
![image](https://github.com/DiferTK/Taller-de-Sakila-BD-2/assets/154281253/0794ee93-37af-46e5-9b83-4d896db55f32)

##6.¿Qué apellidos no se repiten entre diferentes actores?
Esta consulta selecciona los apellidos que son únicos entre los diferentes actores en la tabla 'actor'
```sql
SELECT 
    last_name
FROM 
    actor
GROUP BY 
    last_name
HAVING 
    COUNT(*) = 1;
```
![b8fc4348-8e3f-4132-b096-357b4903d114](https://github.com/DiferTK/Taller-de-Sakila-BD-2/assets/154281253/3baa1e87-f3c7-49c9-9c23-392cfcbcd9c6)


##7.Crear una vista con los 3 géneros que generan mayores ingresos. Enumérelos en orden descendente, considerando el campo 'amount' de la tabla de pagos para el cálculo. 
```sql
create VIEW Top3Generos AS
SELECT 
    category.name AS genre,
    SUM(payment.amount) AS total_revenue
FROM 
    payment
    INNER JOIN rental ON payment.rental_id = rental.rental_id
    INNER JOIN inventory ON rental.inventory_id = inventory.inventory_id
    INNER JOIN film ON inventory.film_id = film.film_id
    INNER JOIN film_category ON film.film_id = film_category.film_id
    INNER JOIN category ON film_category.category_id = category.category_id
GROUP BY 
    category.name
ORDER BY 
    total_revenue DESC
LIMIT 
    3;
SELECT * FROM Top3Generos;
```
##8.Seleccionar las dos películas más vistas en cada ciudad. Esta consulta selecciona las dos películas más vistas en cada ciudad, mostrando el nombre de la ciudad, el título de la película y el recuento de alquileres. ERROR
```sql
SELECT city.city, title, COUNT(rental.rental_id) AS rental_count
FROM rental
INNER JOIN inventory ON rental.inventory_id = inventory.inventory_id
INNER JOIN store ON inventory.store_id = store.store_id
INNER JOIN address ON store.address_id = address.address_id
INNER JOIN city ON address.city_id = city.city_id
INNER JOIN film ON inventory.film_id = film.film_id
GROUP BY city.city, title
ORDER BY city.city, rental_count DESC;
```
##9. Seleccionar el nombre, apellido y correo electrónico de todos los clientes de Estados Unidos que no hayan alquilado ninguna película en los últimos tres meses
```sql
SELECT 
    c.first_name AS nombre,
    c.last_name AS apellido,
    c.email AS correo
FROM 
    customer AS c
WHERE 
    c.address_id IN (
        SELECT 
            a.address_id
        FROM 
            address AS a
            JOIN city AS ci ON a.city_id = ci.city_id
            JOIN country AS co ON ci.country_id = co.country_id
        WHERE 
            co.country = 'United States'
    )
    AND c.customer_id NOT IN (
        SELECT 
            DISTINCT r.customer_id
        FROM 
            rental AS r
        WHERE 
            r.rental_date >= DATE_SUB(NOW(), INTERVAL 3 MONTH)
    );
```

##10. Seleccionar los 3 principales clientes de cada tienda basándose en el número de alquileres realizados. Utilice las funciones Rank, Dense_Rank y Row_Number, y cree un campo booleano adicional que indique los registros donde estas tres funciones devuelvan el mismo valor (0) y los registros donde estas tres funciones no devuelvan el mismo valor (1).

```sql
SELECT 
    *,
    CASE 
        WHEN rank_value = 1 THEN 0
        ELSE 1
    END AS mismo_valor
FROM (
    SELECT 
        *,
        RANK() OVER (PARTITION BY store_id ORDER BY alquileres DESC) AS rank_value
    FROM (
        SELECT 
            s.store_id,
            c.customer_id,
            COUNT(*) AS alquileres
        FROM 
            rental AS r
            JOIN customer AS c ON r.customer_id = c.customer_id
            JOIN store AS s ON c.store_id = s.store_id
        GROUP BY 
            s.store_id,
            c.customer_id
    ) AS rental_counts
) AS ranked_customers
WHERE 
    rank_value <= 3;
```
