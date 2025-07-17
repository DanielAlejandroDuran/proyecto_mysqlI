# **Proyecto MYSQL II Consultas SQL -- Daniel Alejandro Duran Franco**



# Historias de Usuario

## **1. Consultas SQL Especializadas**

1. Como analista, quiero listar todos los productos con su empresa asociada y el precio más bajo por ciudad.

   ```sql
    SELECT
       ->     p.id AS product_id,
       ->     p.name AS product_name,
       ->     c.name AS company_name,
       ->     c.city_id,
       ->     city.name AS city_name,
       ->     MIN(cp.price) AS lowest_price
       -> FROM products p
       -> INNER JOIN companyproducts cp ON p.id = cp.product_id
       -> INNER JOIN companies c ON cp.company_id = c.id
       -> INNER JOIN citiesormunicipalities city ON c.city_id = city.code
       -> GROUP BY p.id, p.name, c.name, c.city_id, city.name
       -> ORDER BY city.name, p.name;
       
   //No puse la tabla porque, era muy grande y no se veia bien//
   ```

   

   2. Como administrador, deseo obtener el top 5 de clientes que más productos han calificado en los últimos 6 meses.

      ```sql
      SELECT 
          c.id AS customer_id,
          c.name AS customer_name,
          c.email,
          COUNT(qp.product_id) AS total_ratings
      FROM customers c
      INNER JOIN quality_products qp ON c.id = qp.customer_id
      WHERE qp.daterating >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
      GROUP BY c.id, c.name, c.email
      ORDER BY total_ratings DESC
      LIMIT 5;
      ```

   3. Como gerente de ventas, quiero ver la distribución de productos por categoría y unidad de medida.

      ```
       SELECT
          ->     cat.id AS category_id,
          ->     cat.description AS category_name,
          ->     um.id AS unit_id,
          ->     um.description AS unit_measure,
          ->     COUNT(p.id) AS total_products,
          ->     ROUND((COUNT(p.id) * 100.0 / (SELECT COUNT(*) FROM products)), 2) AS percentage
          -> FROM categories cat
          -> INNER JOIN products p ON cat.id = p.category_id
          -> INNER JOIN companyproducts cp ON p.id = cp.product_id
          -> INNER JOIN unitofmeasure um ON cp.unitmeasure_id = um.id
          -> GROUP BY cat.id, cat.description, um.id, um.description
          -> ORDER BY cat.description, um.description;
      +-------------+------------------+---------+--------------+----------------+------------+
      | category_id | category_name    | unit_id | unit_measure | total_products | percentage |
      +-------------+------------------+---------+--------------+----------------+------------+
      |           2 | Food & Beverages |       2 | Kilograms    |              1 |      10.00 |
      |           9 | Real Estate      |       9 | Pieces       |              1 |      10.00 |
      |           3 | Retail           |       3 | Liters       |              2 |      20.00 |
      |           1 | Technology       |       1 | Units        |              9 |      90.00 |
      +-------------+------------------+---------+--------------+----------------+------------+
      ```

   4. Como cliente, quiero saber qué productos tienen calificaciones superiores al promedio general.

      ```sql
      SELECT
      ->     p.id AS product_id,
      ->     p.name AS product_name,
      ->     p.price,
      ->     cat.description AS category,
      ->     AVG(qp.rating) AS average_rating,
      ->     COUNT(qp.rating) AS total_ratings
      -> FROM products p
      -> INNER JOIN categories cat ON p.category_id = cat.id
      -> INNER JOIN quality_products qp ON p.id = qp.product_id
      -> GROUP BY p.id, p.name, p.price, cat.description
      -> HAVING AVG(qp.rating) > (
      ->     SELECT AVG(rating)
      ->     FROM quality_products
      -> )
      -> ORDER BY average_rating DESC;
      ```

      5. Como auditor, quiero conocer todas las empresas que no han recibido ninguna calificación.

         ```sql
         SELECT 
             c.id AS company_id,
             c.name AS company_name,
             c.email,
             c.cellphone,
             cat.description AS category,
             city.name AS city_name
         FROM companies c
         INNER JOIN categories cat ON c.category_id = cat.id
         INNER JOIN citiesormunicipalitites city ON c.city_id = city.code
         LEFT JOIN companyproducts cp ON c.id = cp.company_id
         LEFT JOIN quality_products qp ON cp.product_id = qp.product_id
         WHERE qp.product_id IS NULL
         ORDER BY c.name;
         ```

         

      6. Como operador, deseo obtener los productos que han sido añadidos como favoritos por más de 10 clientes distintos.

         ```sql
         SELECT 
             p.id AS product_id,
             p.name AS product_name,
             p.price,
             cat.description AS category,
             COUNT(DISTINCT df.favorite_id) AS total_favorites,
             COUNT(DISTINCT f.customer_id) AS distinct_customers
         FROM products p
         INNER JOIN categories cat ON p.category_id = cat.id
         INNER JOIN details_favorites df ON p.id = df.product_id
         INNER JOIN favorites f ON df.favorite_id = f.id
         GROUP BY p.id, p.name, p.price, cat.description
         HAVING COUNT(DISTINCT f.customer_id) > 10
         ORDER BY distinct_customers DESC;
         ```

      7. Como gerente regional, quiero obtener todas las empresas activas por ciudad y categoría.

         ```sql
         SELECT 
             city.code AS city_code,
             city.name AS city_name,
             cat.id AS category_id,
             cat.description AS category_name,
             COUNT(c.id) AS total_companies,
             GROUP_CONCAT(c.name ORDER BY c.name SEPARATOR ', ') AS companies_list
         FROM companies c
         INNER JOIN categories cat ON c.category_id = cat.id
         INNER JOIN citiesormunicipalitites city ON c.city_id = city.code
         GROUP BY city.code, city.name, cat.id, cat.description
         ORDER BY city.name, cat.description;
         ```

      8. Como especialista en marketing, deseo obtener los 10 productos más calificados en cada ciudad.

         ```sql
         SELECT 
             city_code,
             city_name,
             product_id,
             product_name,
             category_name,
             average_rating,
             total_ratings,
             city_rank
         FROM (
             SELECT 
                 city.code AS city_code,
                 city.name AS city_name,
                 p.id AS product_id,
                 p.name AS product_name,
                 cat.description AS category_name,
                 AVG(qp.rating) AS average_rating,
                 COUNT(qp.rating) AS total_ratings,
                 ROW_NUMBER() OVER (PARTITION BY city.code ORDER BY AVG(qp.rating) DESC) AS city_rank
             FROM products p
             INNER JOIN categories cat ON p.category_id = cat.id
             INNER JOIN companyproducts cp ON p.id = cp.product_id
             INNER JOIN companies c ON cp.company_id = c.id
             INNER JOIN citiesormunicipalities city ON c.city_id = city.code
             INNER JOIN quality_products qp ON p.id = qp.product_id
             GROUP BY city.code, city.name, p.id, p.name, cat.description
         ) ranked_products
         WHERE city_rank <= 10
         ORDER BY city_name, city_rank;
         ```

      9. Como técnico, quiero identificar productos sin unidad de medida asignada.

         ```sql
         SELECT 
             p.id AS product_id,
             p.name AS product_name,
             p.price,
             cat.description AS category_name,
             p.detail
         FROM products p
         INNER JOIN categories cat ON p.category_id = cat.id
         LEFT JOIN companyproducts cp ON p.id = cp.product_id
         WHERE cp.unitmeasure_id IS NULL OR cp.product_id IS NULL
         ORDER BY p.name;
         ```

      10. Como gestor de beneficios, deseo ver los planes de membresía sin beneficios registrados.

          ```sql
          SELECT 
              m.id AS membership_id,
              m.name AS membership_name,
              m.description AS membership_description
          FROM memberships m
          LEFT JOIN membershipbenefits mb ON m.id = mb.membership_id
          WHERE mb.membership_id IS NULL
          ORDER BY m.name;
          
          +---------------+-----------------+----------------------------------------------+
          | membership_id | membership_name | membership_description                       |
          +---------------+-----------------+----------------------------------------------+
          |             7 | Corporate       | Corporate membership for large organizations |
          |             5 | Family          | Family plan for multiple users               |
          |             6 | Professional    | Professional membership for individuals      |
          |             4 | Student         | Special pricing for students                 |
          +---------------+-----------------+----------------------------------------------+
          ```

      11. Como supervisor, quiero obtener los productos de una categoría específica con su promedio de calificación.

          ```sql
          SELECT 
              p.id AS product_id,
              p.name AS product_name,
              p.price,
              cat.description AS category_name,
              COALESCE(AVG(qp.rating), 0) AS average_rating,
              COUNT(qp.rating) AS total_ratings,
              CASE 
                  WHEN COUNT(qp.rating) = 0 THEN 'Sin calificaciones'
                  WHEN AVG(qp.rating) >= 4.5 THEN 'Excelente'
                  WHEN AVG(qp.rating) >= 3.5 THEN 'Bueno'
                  WHEN AVG(qp.rating) >= 2.5 THEN 'Regular'
                  ELSE 'Malo'
              END AS rating_category
          FROM products p
          INNER JOIN categories cat ON p.category_id = cat.id
          LEFT JOIN quality_products qp ON p.id = qp.product_id
          WHERE cat.description = 'NOMBRE_CATEGORIA'
          GROUP BY p.id, p.name, p.price, cat.description
          ORDER BY average_rating DESC, p.name;
          ```

      12. Como asesor, deseo obtener los clientes que han comprado productos de más de una empresa.

          ```sql
          SELECT 
              c.id,
              c.name AS nombre_cliente,
              c.email,
              c.cellphone,
              COUNT(DISTINCT f.company_id) AS total_empresas,
              STRING_AGG(DISTINCT comp.name, ', ') AS empresas
          FROM customers c
          INNER JOIN favorites f ON c.id = f.customer_id
          INNER JOIN companies comp ON f.company_id = comp.id
          GROUP BY c.id, c.name, c.email, c.cellphone
          HAVING COUNT(DISTINCT f.company_id) > 1
          ORDER BY total_empresas DESC, c.name;
          ```

      13. Como director, quiero identificar las ciudades con más clientes activos.

          ```sql
          SELECT 
              c.code AS codigo_ciudad,
              c.name AS nombre_ciudad,
              sr.name AS estado_region,
              co.name AS pais,
              COUNT(cust.id) AS total_clientes,
              ROUND(
                  COUNT(cust.id) * 100.0 / (SELECT COUNT(*) FROM customers), 2
              ) AS porcentaje_clientes
          FROM citiesormunicipalities c
          INNER JOIN customers cust ON c.code = cust.city_id
          INNER JOIN stateorregions sr ON c.statereg_id = sr.code
          INNER JOIN countries co ON sr.country_id = co.isocode
          GROUP BY c.code, c.name, sr.name, co.name
          ORDER BY total_clientes DESC;
          ```

      14. Como analista de calidad, deseo obtener el ranking de productos por empresa basado en la media de `quality_products`.

          ```sql
          SELECT 
              comp.id AS empresa_id,
              comp.name AS nombre_empresa,
              p.id AS producto_id,
              p.name AS nombre_producto,
              cat.description AS categoria,
              COUNT(qp.rating) AS total_evaluaciones,
              ROUND(AVG(qp.rating), 2) AS calidad_promedio,
              MIN(qp.rating) AS calidad_minima,
              MAX(qp.rating) AS calidad_maxima,
              ROW_NUMBER() OVER (
                  PARTITION BY comp.id 
                  ORDER BY AVG(qp.rating) DESC
              ) AS ranking_en_empresa,
              RANK() OVER (
                  ORDER BY AVG(qp.rating) DESC
              ) AS ranking_general
          FROM companies comp
          INNER JOIN products p ON comp.id = p.category_id
          INNER JOIN quality_products qp ON p.id = qp.product_id
          INNER JOIN categories cat ON p.category_id = cat.id
          GROUP BY comp.id, comp.name, p.id, p.name, cat.description
          HAVING COUNT(qp.rating) >= 1
          ORDER BY comp.name, ranking_en_empresa;
          ```

      15. Como administrador, quiero listar empresas que ofrecen más de cinco productos distintos.

          ```sql
          SELECT 
              c.id AS empresa_id,
              c.name AS nombre_empresa,
              c.email,
              c.cellphone,
              COUNT(DISTINCT p.id) AS total_productos,
              COUNT(DISTINCT p.category_id) AS categorias_diferentes,
              ROUND(AVG(p.price), 2) AS precio_promedio,
              MIN(p.price) AS precio_minimo,
              MAX(p.price) AS precio_maximo
          FROM companies c
          INNER JOIN products p ON c.id = p.category_id
          GROUP BY c.id, c.name, c.email, c.cellphone
          HAVING COUNT(DISTINCT p.id) > 5
          ORDER BY total_productos DESC;
          ```

      16. Como cliente, deseo visualizar los productos favoritos que aún no han sido calificados.

          ```sql
          SELECT DISTINCT
              p.id AS product_id,
              p.name AS product_name,
              p.detail AS product_detail,
              p.price AS product_price,
              p.image AS product_image,
              c.name AS category_name,
              comp.name AS company_name,
              f.id AS favorite_id
          FROM favorites f
          INNER JOIN details_favorites df ON f.id = df.favorite_id
          INNER JOIN products p ON df.product_id = p.id
          INNER JOIN categories c ON p.category_id = c.id
          INNER JOIN companies comp ON f.company_id = comp.id
          LEFT JOIN quality_products qp ON (qp.product_id = p.id AND qp.customer_id = f.customer_id)
          WHERE f.customer_id = ?
            AND qp.product_id IS NULL
          ORDER BY p.name;
          ```

      17. Como desarrollador, deseo consultar los beneficios asignados a cada audiencia junto con su descripción.

          ```sql
          SELECT 
              a.id AS audience_id,
              a.description AS audience_description,
              b.id AS benefit_id,
              b.description AS benefit_description,
              b.detail AS benefit_detail
          FROM audiences a
          INNER JOIN audiencebenefits ab ON a.id = ab.audience_id
          INNER JOIN benefits b ON ab.benefit_id = b.id
          ORDER BY a.description, b.description;
          ```

      18. Como operador logístico, quiero saber en qué ciudades hay empresas sin productos asociados.

          ```sql
          SELECT DISTINCT
              cit.code AS city_code,
              cit.name AS city_name,
              sr.name AS state_region_name,
              co.name AS country_name,
              comp.id AS company_id,
              comp.name AS company_name,
              comp.email AS company_email,
              comp.cellphone AS company_phone,
              cat.description AS company_category
          FROM companies comp
          INNER JOIN citiesormunicipalities cit ON comp.city_id = cit.code
          INNER JOIN stateorregions sr ON cit.statereg_id = sr.code
          INNER JOIN countries co ON sr.country_id = co.isocode
          INNER JOIN categories cat ON comp.category_id = cat.id
          LEFT JOIN products p ON comp.id = p.category_id
          WHERE p.id IS NULL
          ORDER BY co.name, sr.name, cit.name, comp.name;
          ```

      19. Como técnico, deseo obtener todas las empresas con productos duplicados por nombre.

          ```sql
          SELECT 
              comp.id AS company_id,
              comp.name AS company_name,
              comp.email AS company_email,
              comp.cellphone AS company_phone,
              p.name AS product_name,
              COUNT(p.id) AS duplicate_count,
              GROUP_CONCAT(p.id ORDER BY p.id SEPARATOR ', ') AS product_ids,
              GROUP_CONCAT(p.price ORDER BY p.id SEPARATOR ', ') AS product_prices
          FROM companies comp
          INNER JOIN companyproducts cp ON comp.id = cp.company_id
          INNER JOIN products p ON cp.product_id = p.id
          GROUP BY comp.id, comp.name, comp.email, comp.cellphone, p.name
          HAVING COUNT(p.id) > 1
          ORDER BY comp.name, p.name;
          ```

      20. Como analista, quiero una vista resumen de clientes, productos favoritos y promedio de calificación recibido.

          ```sql
          SELECT 
              c.id AS customer_id,
              c.name AS customer_name,
              c.email AS customer_email,
              c.cellphone AS customer_phone,
              cit.name AS customer_city,
              sr.name AS customer_state,
              co.name AS customer_country,
              aud.description AS customer_audience,
              
              COUNT(DISTINCT df.product_id) AS total_favorite_products,
              COUNT(DISTINCT f.company_id) AS total_companies_favorited,
              
              COUNT(DISTINCT qp.product_id) AS total_products_rated,
              ROUND(AVG(qp.rating), 2) AS avg_rating_given,
              
              GROUP_CONCAT(
                  DISTINCT CONCAT(p.name, ' ($', p.price, ')') 
                  ORDER BY p.name 
                  SEPARATOR ', '
              ) AS favorite_products_list,
              
              GROUP_CONCAT(
                  DISTINCT cat.description 
                  ORDER BY cat.description 
                  SEPARATOR ', '
              ) AS favorite_categories
          
          FROM customers c
          INNER JOIN citiesormunicipalities cit ON c.city_id = cit.code
          INNER JOIN stateorregions sr ON cit.statereg_id = sr.code
          INNER JOIN countries co ON sr.country_id = co.isocode
          INNER JOIN audiences aud ON c.audience_id = aud.id
          LEFT JOIN favorites f ON c.id = f.customer_id
          LEFT JOIN details_favorites df ON f.id = df.favorite_id
          LEFT JOIN products p ON df.product_id = p.id
          LEFT JOIN categories cat ON p.category_id = cat.id
          LEFT JOIN quality_products qp ON c.id = qp.customer_id
          
          GROUP BY 
              c.id, c.name, c.email, c.cellphone, 
              cit.name, sr.name, co.name, aud.description
          
          ORDER BY total_favorite_products DESC, avg_rating_given DESC;
          ```

      

      ##  **2. Subconsultas**

      1. Como gerente, quiero ver los productos cuyo precio esté por encima del promedio de su categoría.

         ```sql
         SELECT 
             p.id AS product_id,
             p.name AS product_name,
             p.detail AS product_detail,
             p.price AS product_price,
             p.image AS product_image,
             c.id AS category_id,
             c.description AS category_name,
             ROUND(cat_avg.avg_price, 2) AS category_avg_price,
             ROUND(p.price - cat_avg.avg_price, 2) AS price_difference,
             ROUND(((p.price - cat_avg.avg_price) / cat_avg.avg_price) * 100, 2) AS percentage_above_avg
         FROM products p
         INNER JOIN categories c ON p.category_id = c.id
         INNER JOIN (
             SELECT 
                 category_id,
                 AVG(price) AS avg_price
             FROM products
             GROUP BY category_id
         ) cat_avg ON p.category_id = cat_avg.category_id
         WHERE p.price > cat_avg.avg_price
         ORDER BY c.description, percentage_above_avg DESC;
         ```

      2. Como administrador, deseo listar las empresas que tienen más productos que la media de empresas.

         ```sql
         SELECT 
             comp.id AS company_id,
             comp.name AS company_name,
             comp.email AS company_email,
             comp.cellphone AS company_phone,
             cat.description AS company_category,
             cit.name AS city_name,
             sr.name AS state_name,
             co.name AS country_name,
             
             COUNT(DISTINCT cp.product_id) AS total_products,
             company_avg.avg_products_per_company,
             ROUND(COUNT(DISTINCT cp.product_id) - company_avg.avg_products_per_company, 2) AS products_above_avg,
             ROUND(((COUNT(DISTINCT cp.product_id) - company_avg.avg_products_per_company) / company_avg.avg_products_per_company) * 100, 2) AS percentage_above_avg,
             
             ROUND(AVG(p.price), 2) AS avg_product_price,
             MIN(p.price) AS min_product_price,
             MAX(p.price) AS max_product_price,
             COUNT(DISTINCT p.category_id) AS product_categories_count
         
         FROM companies comp
         INNER JOIN companyproducts cp ON comp.id = cp.company_id
         INNER JOIN products p ON cp.product_id = p.id
         INNER JOIN categories cat ON comp.category_id = cat.id
         INNER JOIN citiesormunicipalities cit ON comp.city_id = cit.code
         INNER JOIN stateorregions sr ON cit.statereg_id = sr.code
         INNER JOIN countries co ON sr.country_id = co.isocode
         CROSS JOIN (
             SELECT 
                 AVG(product_count) AS avg_products_per_company
             FROM (
                 SELECT 
                     company_id,
                     COUNT(product_id) AS product_count
                 FROM companyproducts
                 GROUP BY company_id
             ) AS company_product_counts
         ) company_avg
         
         GROUP BY 
             comp.id, comp.name, comp.email, comp.cellphone, 
             cat.description, cit.name, sr.name, co.name,
             company_avg.avg_products_per_company
         
         HAVING COUNT(DISTINCT cp.product_id) > company_avg.avg_products_per_company
         
         ORDER BY total_products DESC, percentage_above_avg DESC;
         ```

      3. Como cliente, quiero ver mis productos favoritos que han sido calificados por otros clientes.

         ```sql
         SELECT 
             p.id AS product_id,
             p.name AS product_name,
             p.detail AS product_detail,
             p.price AS product_price,
             p.image AS product_image,
             cat.description AS category_name,
             comp.name AS company_name,
             comp.email AS company_email,
             
             COUNT(DISTINCT qp.customer_id) AS total_customers_rated,
             ROUND(AVG(qp.rating), 2) AS avg_rating,
             MIN(qp.rating) AS min_rating,
             MAX(qp.rating) AS max_rating,
             
             SUM(CASE WHEN qp.rating = 5 THEN 1 ELSE 0 END) AS rating_5_count,
             SUM(CASE WHEN qp.rating = 4 THEN 1 ELSE 0 END) AS rating_4_count,
             SUM(CASE WHEN qp.rating = 3 THEN 1 ELSE 0 END) AS rating_3_count,
             SUM(CASE WHEN qp.rating = 2 THEN 1 ELSE 0 END) AS rating_2_count,
             SUM(CASE WHEN qp.rating = 1 THEN 1 ELSE 0 END) AS rating_1_count,
             
             CASE 
                 WHEN AVG(qp.rating) >= 4.5 THEN 'Excelente'
                 WHEN AVG(qp.rating) >= 3.5 THEN 'Muy Bueno'
                 WHEN AVG(qp.rating) >= 2.5 THEN 'Bueno'
                 WHEN AVG(qp.rating) >= 1.5 THEN 'Regular'
                 ELSE 'Malo'
             END AS product_rating_category,
             
             MAX(qp.daterating) AS last_rating_date
         
         FROM favorites f
         INNER JOIN details_favorites df ON f.id = df.favorite_id
         INNER JOIN products p ON df.product_id = p.id
         INNER JOIN categories cat ON p.category_id = cat.id
         INNER JOIN companies comp ON f.company_id = comp.id
         INNER JOIN quality_products qp ON p.id = qp.product_id
         WHERE f.customer_id = ?
           AND qp.customer_id != ?
         GROUP BY 
             p.id, p.name, p.detail, p.price, p.image,
             cat.description, comp.name, comp.email
         HAVING COUNT(DISTINCT qp.customer_id) > 0
         ORDER BY avg_rating DESC, total_customers_rated DESC;
         ```

      4. Como supervisor, deseo obtener los productos con el mayor número de veces añadidos como favoritos.

         ```sql
         SELECT 
             p.id AS product_id,
             p.name AS product_name,
             p.detail AS product_detail,
             p.price AS product_price,
             p.image AS product_image,
             cat.description AS category_name,
             
             COUNT(DISTINCT df.favorite_id) AS total_favorites,
             COUNT(DISTINCT f.customer_id) AS unique_customers,
             COUNT(DISTINCT f.company_id) AS companies_offering,
             
             GROUP_CONCAT(DISTINCT comp.name ORDER BY comp.name SEPARATOR ', ') AS company_names,
             
             COUNT(DISTINCT qp.customer_id) AS customers_rated,
             ROUND(AVG(qp.rating), 2) AS avg_rating,
             
             RANK() OVER (ORDER BY COUNT(DISTINCT df.favorite_id) DESC) AS popularity_rank,
             
             CASE 
                 WHEN COUNT(DISTINCT df.favorite_id) >= 50 THEN 'Muy Popular'
                 WHEN COUNT(DISTINCT df.favorite_id) >= 20 THEN 'Popular'
                 WHEN COUNT(DISTINCT df.favorite_id) >= 10 THEN 'Moderadamente Popular'
                 WHEN COUNT(DISTINCT df.favorite_id) >= 5 THEN 'Poco Popular'
                 ELSE 'Nuevo/Nicho'
             END AS popularity_level,
             
             CASE 
                 WHEN AVG(qp.rating) IS NULL THEN 'Sin calificar'
                 WHEN AVG(qp.rating) >= 4.0 AND COUNT(DISTINCT df.favorite_id) >= 20 THEN 'Éxito Total'
                 WHEN AVG(qp.rating) >= 4.0 THEN 'Bien Calificado'
                 WHEN COUNT(DISTINCT df.favorite_id) >= 20 THEN 'Popular pero Regular'
                 ELSE 'Promedio'
             END AS success_category
         
         FROM products p
         INNER JOIN categories cat ON p.category_id = cat.id
         INNER JOIN details_favorites df ON p.id = df.product_id
         INNER JOIN favorites f ON df.favorite_id = f.id
         INNER JOIN companies comp ON f.company_id = comp.id
         LEFT JOIN quality_products qp ON p.id = qp.product_id
         
         GROUP BY 
             p.id, p.name, p.detail, p.price, p.image, cat.description
         
         ORDER BY total_favorites DESC, avg_rating DESC;
         ```

      5. Como técnico, quiero listar los clientes cuyo correo no aparece en la tabla `rates` ni en `quality_products`.

         ```sql
         SELECT 
             c.id AS customer_id,
             c.name AS customer_name,
             c.email AS customer_email,
             c.cellphone AS customer_phone,
             c.address AS customer_address,
             cit.name AS city_name,
             sr.name AS state_name,
             co.name AS country_name,
             aud.description AS audience_type,
             
             COUNT(DISTINCT f.id) AS total_favorites,
             
             CASE 
                 WHEN COUNT(DISTINCT f.id) > 0 THEN 'Con Favoritos - Sin Calificar'
                 ELSE 'Inactivo Total'
             END AS customer_status,
             
             CASE 
                 WHEN c.email IS NULL OR c.email = '' THEN 'Sin Email'
                 ELSE 'Email Válido'
             END AS email_status
         
         FROM customers c
         INNER JOIN citiesormunicipalities cit ON c.city_id = cit.code
         INNER JOIN stateorregions sr ON cit.statereg_id = sr.code
         INNER JOIN countries co ON sr.country_id = co.isocode
         INNER JOIN audiences aud ON c.audience_id = aud.id
         LEFT JOIN favorites f ON c.id = f.customer_id
         LEFT JOIN rates r ON c.id = r.customer_id
         LEFT JOIN quality_products qp ON c.id = qp.customer_id
         
         WHERE r.customer_id IS NULL 
           AND qp.customer_id IS NULL
           AND c.email IS NOT NULL 
           AND c.email != ''
         
         GROUP BY 
             c.id, c.name, c.email, c.cellphone, c.address,
             cit.name, sr.name, co.name, aud.description
         
         ORDER BY customer_status, c.name;
         ```

      6. Como gestor de calidad, quiero obtener los productos con una calificación inferior al mínimo de su categoría.

         ```sql
         SELECT 
             p.id AS product_id,
             p.name AS product_name,
             p.detail AS product_detail,
             p.price AS product_price,
             p.image AS product_image,
             cat.description AS category_name,
             
             COUNT(DISTINCT qp.customer_id) AS total_ratings,
             ROUND(AVG(qp.rating), 2) AS avg_product_rating,
             MIN(qp.rating) AS min_product_rating,
             MAX(qp.rating) AS max_product_rating,
             
             cat_stats.min_category_rating,
             cat_stats.avg_category_rating,
             cat_stats.max_category_rating,
             
             ROUND(AVG(qp.rating) - cat_stats.min_category_rating, 2) AS difference_from_category_min,
             ROUND(AVG(qp.rating) - cat_stats.avg_category_rating, 2) AS difference_from_category_avg,
             
             GROUP_CONCAT(DISTINCT comp.name ORDER BY comp.name SEPARATOR ', ') AS companies_offering,
             
             CASE 
                 WHEN AVG(qp.rating) < cat_stats.min_category_rating - 1 THEN 'Crítico'
                 WHEN AVG(qp.rating) < cat_stats.min_category_rating - 0.5 THEN 'Muy Bajo'
                 WHEN AVG(qp.rating) < cat_stats.min_category_rating THEN 'Bajo'
                 ELSE 'Aceptable'
             END AS quality_alert_level,
             
             MIN(qp.daterating) AS first_rating_date,
             MAX(qp.daterating) AS last_rating_date
         
         FROM products p
         INNER JOIN categories cat ON p.category_id = cat.id
         INNER JOIN quality_products qp ON p.id = qp.product_id
         INNER JOIN (
             SELECT 
                 p2.category_id,
                 MIN(avg_rating) AS min_category_rating,
                 AVG(avg_rating) AS avg_category_rating,
                 MAX(avg_rating) AS max_category_rating
             FROM (
                 SELECT 
                     p3.id,
                     p3.category_id,
                     AVG(qp2.rating) AS avg_rating
                 FROM products p3
                 INNER JOIN quality_products qp2 ON p3.id = qp2.product_id
                 GROUP BY p3.id, p3.category_id
             ) AS product_ratings
             GROUP BY category_id
         ) cat_stats ON p.category_id = cat_stats.category_id
         LEFT JOIN companyproducts cp ON p.id = cp.product_id
         LEFT JOIN companies comp ON cp.company_id = comp.id
         
         GROUP BY 
             p.id, p.name, p.detail, p.price, p.image, cat.description,
             cat_stats.min_category_rating, cat_stats.avg_category_rating, cat_stats.max_category_rating
         
         HAVING AVG(qp.rating) < cat_stats.min_category_rating
         
         ORDER BY difference_from_category_min ASC, avg_product_rating ASC;
         ```

      7. Como desarrollador, deseo listar las ciudades que no tienen clientes registrados.

         ```sql
         SELECT 
             cit.code AS city_code,
             cit.name AS city_name,
             sr.code AS state_code,
             sr.name AS state_name,
             co.isocode AS country_code,
             co.name AS country_name,
             co.alfaisotwo AS country_iso2,
             co.alfaisothree AS country_iso3,
             
             COUNT(DISTINCT comp.id) AS companies_count,
             
             CASE 
                 WHEN COUNT(DISTINCT comp.id) > 0 THEN 'Con Empresas'
                 ELSE 'Sin Empresas'
             END AS business_presence,
             
             GROUP_CONCAT(DISTINCT comp.name ORDER BY comp.name SEPARATOR ', ') AS companies_list,
             
             CASE 
                 WHEN COUNT(DISTINCT comp.id) > 5 THEN 'Alta Oportunidad'
                 WHEN COUNT(DISTINCT comp.id) > 2 THEN 'Oportunidad Media'
                 WHEN COUNT(DISTINCT comp.id) > 0 THEN 'Oportunidad Baja'
                 ELSE 'Sin Oportunidad Empresarial'
             END AS market_opportunity
         
         FROM citiesormunicipalities cit
         INNER JOIN stateorregions sr ON cit.statereg_id = sr.code
         INNER JOIN countries co ON sr.country_id = co.isocode
         LEFT JOIN customers c ON cit.code = c.city_id
         LEFT JOIN companies comp ON cit.code = comp.city_id
         
         WHERE c.city_id IS NULL
         
         GROUP BY 
             cit.code, cit.name, sr.code, sr.name, 
             co.isocode, co.name, co.alfaisotwo, co.alfaisothree
         
         ORDER BY co.name, sr.name, cit.name;
         
         SELECT 
             co.name AS country_name,
             sr.name AS state_name,
             COUNT(DISTINCT cit.code) AS cities_without_customers,
             COUNT(DISTINCT comp.id) AS total_companies_in_empty_cities,
             
             (SELECT COUNT(DISTINCT city_id) FROM customers c2 
              INNER JOIN citiesormunicipalities cit2 ON c2.city_id = cit2.code 
              WHERE cit2.statereg_id = sr.code) AS cities_with_customers,
             
             ROUND(
                 (COUNT(DISTINCT cit.code) * 100.0) / 
                 (SELECT COUNT(*) FROM citiesormunicipalities cit3 WHERE cit3.statereg_id = sr.code),
                 2
             ) AS empty_cities_percentage,
             
             MAX(company_counts.companies_per_city) AS max_companies_in_empty_city,
             
             CASE 
                 WHEN COUNT(DISTINCT cit.code) = 0 THEN 'Totalmente Cubierto'
                 WHEN COUNT(DISTINCT cit.code) <= 2 THEN 'Bien Cubierto'
                 WHEN COUNT(DISTINCT cit.code) <= 5 THEN 'Moderadamente Cubierto'
                 ELSE 'Poco Cubierto'
             END AS coverage_level
         
         FROM citiesormunicipalities cit
         INNER JOIN stateorregions sr ON cit.statereg_id = sr.code
         INNER JOIN countries co ON sr.country_id = co.isocode
         LEFT JOIN customers c ON cit.code = c.city_id
         LEFT JOIN companies comp ON cit.code = comp.city_id
         LEFT JOIN (
             SELECT 
                 city_id,
                 COUNT(*) AS companies_per_city
             FROM companies
             GROUP BY city_id
         ) company_counts ON cit.code = company_counts.city_id
         
         WHERE c.city_id IS NULL
         
         GROUP BY co.name, sr.name, sr.code
         
         ORDER BY empty_cities_percentage DESC, cities_without_customers DESC;
         ```

      8. Como administrador, quiero ver los productos que no han sido evaluados en ninguna encuesta.

         ```sql
         SELECT 
             p.id,
             p.name,
             p.detail,
             p.price,
             c.description as category,
             p.image
         FROM products p
         LEFT JOIN categories c ON p.category_id = c.id
         WHERE p.id NOT IN (
             SELECT DISTINCT product_id 
             FROM quality_products 
             WHERE product_id IS NOT NULL
         )
         ORDER BY p.name;
         ```

      9. Como auditor, quiero listar los beneficios que no están asignados a ninguna audiencia.

         ```sql
         SELECT 
             b.id,
             b.description,
             b.detail
         FROM benefits b
         WHERE b.id NOT IN (
             SELECT DISTINCT benefit_id 
             FROM audiencebenefits 
             WHERE benefit_id IS NOT NULL
         )
         ORDER BY b.description;
         ```

      10. Como cliente, deseo obtener mis productos favoritos que no están disponibles actualmente en ninguna empresa.

          ```sql
          SELECT 
              p.id,
              p.name,
              p.detail,
              p.price,
              c.description as category,
              p.image,
              cust.name as customer_name
          FROM favorites f
          INNER JOIN products p ON f.company_id = p.id
          INNER JOIN customers cust ON f.customer_id = cust.id
          LEFT JOIN categories c ON p.category_id = c.id
          WHERE f.customer_id = @customer_id
              AND p.id NOT IN (
                  SELECT DISTINCT product_id 
                  FROM companyproducts 
                  WHERE product_id IS NOT NULL
              )
          ORDER BY p.name;
          ```

      11. Como director, deseo consultar los productos vendidos en empresas cuya ciudad tenga menos de tres empresas registradas.

          ```sql
          SELECT 
              p.id,
              p.name,
              p.detail,
              p.price,
              cat.description as category,
              p.image,
              comp.name as company_name,
              comp.email as company_email,
              comp.cellphone as company_phone,
              city.name as city_name,
              cp.price as company_price,
              um.description as unit_of_measure
          FROM companyproducts cp
          INNER JOIN products p ON cp.product_id = p.id
          INNER JOIN companies comp ON cp.company_id = comp.id
          INNER JOIN citiesormunicipalities city ON comp.city_id = city.code
          LEFT JOIN categories cat ON p.category_id = cat.id
          LEFT JOIN unitofmeasure um ON cp.unitmeasure_id = um.id
          WHERE comp.city_id IN (
              SELECT city_id
              FROM companies
              GROUP BY city_id
              HAVING COUNT(*) < 3
          )
          ORDER BY city.name, comp.name, p.name;
          ```

      12. Como analista, quiero ver los productos con calidad superior al promedio de todos los productos.

          ```sql
          SELECT 
              p.id,
              p.name,
              p.detail,
              p.price,
              cat.description as category,
              p.image,
              AVG(qp.rating) as average_rating,
              COUNT(qp.rating) as total_evaluations,
              (SELECT AVG(rating) FROM quality_products WHERE rating IS NOT NULL) as global_average
          FROM products p
          INNER JOIN quality_products qp ON p.id = qp.product_id
          LEFT JOIN categories cat ON p.category_id = cat.id
          GROUP BY p.id, p.name, p.detail, p.price, cat.description, p.image
          HAVING AVG(qp.rating) > (
              SELECT AVG(rating) 
              FROM quality_products 
              WHERE rating IS NOT NULL
          )
          ORDER BY AVG(qp.rating) DESC;
          ```

      13. Como gestor, quiero ver empresas que sólo venden productos de una única categoría.

          ```sql
          SELECT 
              c.id AS empresa_id,
              c.name AS nombre_empresa,
              c.email AS email_empresa,
              c.cellphone AS telefono_empresa,
              cat.description AS categoria_unica,
              COUNT(DISTINCT p.id) AS total_productos,
              STRING_AGG(DISTINCT p.name, ', ') AS lista_productos,
              ROUND(AVG(p.price), 2) AS precio_promedio
          FROM companies c
          INNER JOIN products p ON c.id = p.category_id
          INNER JOIN categories cat ON p.category_id = cat.id
          GROUP BY c.id, c.name, c.email, c.cellphone, cat.id, cat.description
          HAVING COUNT(DISTINCT p.category_id) = 1
          ORDER BY total_productos DESC, c.name;
          ```
      
      14. Como gerente comercial, quiero consultar los productos con el mayor precio entre todas las empresas.
      
          ```sql
          SELECT 
              p.id AS producto_id,
              p.name AS nombre_producto,
              p.detail AS descripcion_producto,
              p.price AS precio,
              p.image AS imagen_producto,
              cat.description AS categoria,
              c.name AS nombre_empresa,
              c.email AS email_empresa,
              c.cellphone AS telefono_empresa,
              aud.description AS audiencia_objetivo
          FROM products p
          INNER JOIN categories cat ON p.category_id = cat.id
          INNER JOIN companies c ON c.category_id = cat.id
          LEFT JOIN audiences aud ON c.audience_id = aud.id
          WHERE p.price = (SELECT MAX(price) FROM products)
          ORDER BY p.name;
          ```
      
      15. Como cliente, quiero saber si algún producto de mis favoritos ha sido calificado por otro cliente con más de 4 estrellas.
      
          ```sql
          DECLARE @customer_id INT = 1;
          
          SELECT DISTINCT
              p.id AS producto_id,
              p.name AS nombre_producto,
              p.detail AS descripcion_producto,
              p.price AS precio,
              cat.description AS categoria,
              comp.name AS nombre_empresa,
              COUNT(qp.rating) AS total_calificaciones_altas,
              ROUND(AVG(qp.rating), 2) AS calificacion_promedio,
              MAX(qp.rating) AS calificacion_maxima,
              MIN(qp.rating) AS calificacion_minima
          FROM favorites f
          INNER JOIN products p ON f.company_id = p.id
          INNER JOIN quality_products qp ON p.id = qp.product_id
          INNER JOIN categories cat ON p.category_id = cat.id
          INNER JOIN companies comp ON f.company_id = comp.id
          WHERE f.customer_id = @customer_id
            AND qp.customer_id != @customer_id
            AND qp.rating > 4
          GROUP BY p.id, p.name, p.detail, p.price, cat.description, comp.name
          ORDER BY calificacion_promedio DESC, total_calificaciones_altas DESC;
          ```
      
      16. Como operador, quiero saber qué productos no tienen imagen asignada pero sí han sido calificados.
      
          ```sql
          SELECT DISTINCT
              p.id AS producto_id,
              p.name AS nombre_producto,
              p.detail AS descripcion_producto,
              p.price AS precio,
              p.image AS imagen_producto,
              cat.description AS categoria,
              COUNT(qp.rating) AS total_calificaciones,
              ROUND(AVG(qp.rating), 2) AS calificacion_promedio,
              MIN(qp.rating) AS calificacion_minima,
              MAX(qp.rating) AS calificacion_maxima,
              MIN(qp.daterating) AS primera_calificacion,
              MAX(qp.daterating) AS ultima_calificacion
          FROM products p
          INNER JOIN quality_products qp ON p.id = qp.product_id
          INNER JOIN categories cat ON p.category_id = cat.id
          WHERE (p.image IS NULL OR p.image = '' OR TRIM(p.image) = '')
          GROUP BY p.id, p.name, p.detail, p.price, p.image, cat.description
          ORDER BY total_calificaciones DESC, calificacion_promedio DESC;
          ```
      
      17. Como auditor, quiero ver los planes de membresía sin periodo vigente.
      
          ```sql
          SELECT 
              m.id AS membresia_id,
              m.name AS nombre_membresia,
              m.description AS descripcion_membresia,
              CASE 
                  WHEN NOT EXISTS (
                      SELECT 1 FROM membershipperiods mp 
                      WHERE mp.membership_id = m.id
                  ) THEN 'SIN PERIODOS ASIGNADOS'
                  ELSE 'TODOS LOS PERIODOS VENCIDOS'
              END AS estado_membresia,
              COALESCE(
                  (SELECT COUNT(*) FROM membershipperiods mp2 
                   WHERE mp2.membership_id = m.id), 0
              ) AS total_periodos_historicos
          FROM memberships m
          WHERE m.id NOT IN (
              SELECT DISTINCT mp.membership_id 
              FROM membershipperiods mp 
              INNER JOIN periods p ON mp.period_id = p.id
              WHERE CURRENT_DATE BETWEEN 
                  CAST(p.name AS DATE) AND 
                  DATEADD(MONTH, 1, CAST(p.name AS DATE))
          )
          ORDER BY m.name;
          
          ```
      
      18. Como especialista, quiero identificar los beneficios compartidos por más de una audiencia.
      
          ```sql
          SELECT 
              b.id AS beneficio_id,
              b.description AS descripcion_beneficio,
              b.detail AS detalle_beneficio,
              COUNT(DISTINCT mb.audience_id) AS total_audiencias,
              STRING_AGG(DISTINCT a.description, ', ') AS audiencias_que_comparten,
              COUNT(DISTINCT mb.membership_id) AS total_membresias,
              STRING_AGG(DISTINCT m.name, ', ') AS membresias_asociadas
          FROM benefits b
          INNER JOIN membershipbenefits mb ON b.id = mb.benefit_id
          INNER JOIN audiences a ON mb.audience_id = a.id
          INNER JOIN memberships m ON mb.membership_id = m.id
          GROUP BY b.id, b.description, b.detail
          HAVING COUNT(DISTINCT mb.audience_id) > 1
          ORDER BY total_audiencias DESC, total_membresias DESC;
          ```
      
      19. Como técnico, quiero encontrar empresas cuyos productos no tengan unidad de medida definida.
      
          ```sql
          SELECT DISTINCT
              comp.id AS empresa_id,
              comp.name AS nombre_empresa,
              comp.email AS email_empresa,
              comp.cellphone AS telefono_empresa,
              cat_comp.description AS categoria_empresa,
              aud.description AS audiencia_objetivo,
              COUNT(DISTINCT p.id) AS total_productos_sin_unidad,
              STRING_AGG(DISTINCT p.name, ', ') AS productos_sin_unidad
          FROM companies comp
          INNER JOIN companyproducts cp ON comp.id = cp.company_id
          INNER JOIN products p ON cp.product_id = p.id
          LEFT JOIN categories cat_comp ON comp.category_id = cat_comp.id
          LEFT JOIN audiences aud ON comp.audience_id = aud.id
          LEFT JOIN unitofmeasure um ON cp.unitmeasure_id = um.id
          WHERE cp.unitmeasure_id IS NULL OR um.id IS NULL
          GROUP BY comp.id, comp.name, comp.email, comp.cellphone, cat_comp.description, aud.description
          ORDER BY total_productos_sin_unidad DESC, comp.name;
          ```
      
      20. Como gestor de campañas, deseo obtener los clientes con membresía activa y sin productos favoritos.
      
          ```sql
          SELECT DISTINCT
              c.id AS cliente_id,
              c.name AS nombre_cliente,
              c.email AS email_cliente,
              c.cellphone AS telefono_cliente,
              c.address AS direccion_cliente,
              city.name AS ciudad,
              state.name AS estado,
              country.name AS pais,
              m.name AS nombre_membresia,
              m.description AS descripcion_membresia,
              p.name AS periodo_actual,
              mp.price AS precio_membresia,
              aud.description AS audiencia_membresia
          FROM customers c
          INNER JOIN citiesormunicipalities city ON c.city_id = city.code
          INNER JOIN stateorregions state ON city.statereg_id = state.code
          INNER JOIN countries country ON state.country_id = country.isocode
          INNER JOIN audiences aud ON c.audience_id = aud.id
          INNER JOIN membershipbenefits mb ON aud.id = mb.audience_id
          INNER JOIN memberships m ON mb.membership_id = m.id
          INNER JOIN membershipperiods mp ON m.id = mp.membership_id
          INNER JOIN periods p ON mp.period_id = p.id
          WHERE 
              CURRENT_DATE BETWEEN 
                  CAST(p.name AS DATE) AND 
                  DATEADD(MONTH, 1, CAST(p.name AS DATE))
              AND c.id NOT IN (
                  SELECT DISTINCT customer_id 
                  FROM favorites 
                  WHERE customer_id IS NOT NULL
              )
          ORDER BY m.name, c.name;
          ```
      
      
      
      ##  **3. Funciones Agregadas**
      
      ### **1. Obtener el promedio de calificación por producto**
      
      ```sql
      SELECT 
          p.id AS producto_id,
          p.name AS nombre_producto,
          p.detail AS descripcion_producto,
          p.price AS precio,
          cat.description AS categoria,
          COUNT(qp.rating) AS total_calificaciones,
          ROUND(AVG(qp.rating), 2) AS promedio_calificacion,
          MIN(qp.rating) AS calificacion_minima,
          MAX(qp.rating) AS calificacion_maxima,
          ROUND(STDDEV(qp.rating), 2) AS desviacion_estandar,
          MIN(qp.daterating) AS primera_calificacion,
          MAX(qp.daterating) AS ultima_calificacion
      FROM products p
      INNER JOIN quality_products qp ON p.id = qp.product_id
      INNER JOIN categories cat ON p.category_id = cat.id
      GROUP BY p.id, p.name, p.detail, p.price, cat.description
      ORDER BY promedio_calificacion DESC, total_calificaciones DESC;
      ```
      
      ### **2. Contar cuántos productos ha calificado cada cliente**
      
      ```sql
      SELECT 
          c.id AS cliente_id,
          c.name AS nombre_cliente,
          c.email AS email_cliente,
          c.cellphone AS telefono_cliente,
          city.name AS ciudad,
          state.name AS estado,
          aud.description AS audiencia,
          COUNT(DISTINCT qp.product_id) AS total_productos_calificados,
          COUNT(qp.rating) AS total_calificaciones,
          ROUND(AVG(qp.rating), 2) AS promedio_calificaciones_dadas,
          MIN(qp.rating) AS calificacion_minima_dada,
          MAX(qp.rating) AS calificacion_maxima_dada,
          MIN(qp.daterating) AS primera_calificacion,
          MAX(qp.daterating) AS ultima_calificacion
      FROM customers c
      INNER JOIN quality_products qp ON c.id = qp.customer_id
      INNER JOIN citiesormunicipalities city ON c.city_id = city.code
      INNER JOIN stateorregions state ON city.statereg_id = state.code
      INNER JOIN audiences aud ON c.audience_id = aud.id
      GROUP BY c.id, c.name, c.email, c.cellphone, city.name, state.name, aud.description
      ORDER BY total_productos_calificados DESC, total_calificaciones DESC;
      ```
      
      ### **3. Sumar el total de beneficios asignados por audiencia**
      
      ```sql
      SELECT 
          a.id AS audiencia_id,
          a.description AS audiencia,
          COUNT(DISTINCT mb.benefit_id) AS total_beneficios_unicos,
          COUNT(mb.benefit_id) AS total_asignaciones_beneficios,
          COUNT(DISTINCT mb.membership_id) AS total_membresias,
          COUNT(DISTINCT mb.period_id) AS total_periodos,
          STRING_AGG(DISTINCT b.description, ', ') AS lista_beneficios
      FROM audiences a
      INNER JOIN membershipbenefits mb ON a.id = mb.audience_id
      INNER JOIN benefits b ON mb.benefit_id = b.id
      GROUP BY a.id, a.description
      ORDER BY total_beneficios_unicos DESC, total_asignaciones_beneficios DESC;
      ```
      
      ### **4. Calcular la media de productos por empresa**
      
      ```sql
      SELECT AVG(productos_por_empresa) as media_productos_por_empresa
      FROM (
          SELECT COUNT(cp.product_id) as productos_por_empresa
          FROM companies c
          LEFT JOIN companyproducts cp ON c.id = cp.company_id
          GROUP BY c.id
      ) as subquery;
      
      +-----------------------------+
      | media_productos_por_empresa |
      +-----------------------------+
      |                      1.4444 |
      +-----------------------------+
      ```
      
      ### **5. Contar el total de empresas por ciudad**
      
      ```sql
      SELECT 
          cm.name as ciudad,
          COUNT(c.id) as total_empresas
      FROM citiesormunicipalities cm
      LEFT JOIN companies c ON cm.code = c.city_id
      GROUP BY cm.code, cm.name
      ORDER BY total_empresas DESC;
      ```
      
      ### **6. Calcular el promedio de precios por unidad de medida**
      
      ```sql
      SELECT 
          um.description as unidad_medida,
          COUNT(cp.product_id) as total_productos,
          AVG(cp.price) as precio_promedio,
          MIN(cp.price) as precio_minimo,
          MAX(cp.price) as precio_maximo
      FROM unitofmeasure um
      LEFT JOIN companyproducts cp ON um.id = cp.unitmeasure_id
      GROUP BY um.id, um.description
      ORDER BY precio_promedio DESC;
      ```
      
      ### **7. Contar cuántos clientes hay por ciudad**
      
      ```sql
      SELECT 
          cm.name as ciudad,
          COUNT(c.id) as total_clientes
      FROM citiesormunicipalities cm
      LEFT JOIN customers c ON cm.code = c.city_id
      GROUP BY cm.code, cm.name
      ORDER BY total_clientes DESC;
      ```
      
      ### **8. Calcular planes de membresía por periodo**
      
      ```sql
      SELECT 
          p.name as periodo,
          COUNT(mp.membership_id) as total_membresías,
          AVG(mp.price) as precio_promedio,
          MIN(mp.price) as precio_minimo,
          MAX(mp.price) as precio_maximo
      FROM periods p
      LEFT JOIN membershipperiods mp ON p.id = mp.period_id
      GROUP BY p.id, p.name
      ORDER BY total_membresías DESC;
      ```
      
      ### **9. Ver el promedio de calificaciones dadas por un cliente a sus favoritos**
      
      ```sql
      SELECT 
          c.name as cliente,
          COUNT(qp.rating) as total_calificaciones,
          AVG(qp.rating) as calificación_promedio,
          MIN(qp.rating) as calificación_minima,
          MAX(qp.rating) as calificación_maxima
      FROM customers c
      LEFT JOIN quality_products qp ON c.id = qp.customer_id
      GROUP BY c.id, c.name
      HAVING COUNT(qp.rating) > 0
      ORDER BY calificación_promedio DESC;
      ```
      
      ### **10. Consultar la fecha más reciente en que se calificó un producto**
      
      ```sql
      SELECT 
          MAX(daterating) as fecha_mas_reciente
      FROM quality_products;
      
      +---------------------+
      | fecha_mas_reciente  |
      +---------------------+
      | 2024-01-24 08:20:00 |
      +---------------------+
      ```
      
      ### **11. Obtener la desviación estándar de precios por categoría**
      
      ```sql
      SELECT 
          cat.description as categoría,
          COUNT(cp.price) as total_productos,
          AVG(cp.price) as precio_promedio,
          STDDEV(cp.price) as desviacion_estandar,
          VARIANCE(cp.price) as varianza,
          MIN(cp.price) as precio_minimo,
          MAX(cp.price) as precio_maximo
      FROM categories cat
      LEFT JOIN products p ON cat.id = p.category_id
      LEFT JOIN companyproducts cp ON p.id = cp.product_id
      GROUP BY cat.id, cat.description
      HAVING COUNT(cp.price) > 0
      ORDER BY desviacion_estandar DESC;
      ```
      
      ### **12. Contar cuántas veces un producto fue favorito**
      
      ```sql
      SELECT 
          p.name as producto,
          COUNT(df.favorite_id) as veces_favorito,
          cat.description as categoría
      FROM products p
      LEFT JOIN details_favorites df ON p.id = df.product_id
      LEFT JOIN categories cat ON p.category_id = cat.id
      GROUP BY p.id, p.name, cat.description
      ORDER BY veces_favorito DESC;
      ```
      
      ### **13. Calcular el porcentaje de productos evaluados**
      
      ```sql
      SELECT 
          COUNT(DISTINCT p.id) as total_productos,
          COUNT(DISTINCT qp.product_id) as productos_evaluados,
          ROUND(
              (COUNT(DISTINCT qp.product_id) * 100.0 / COUNT(DISTINCT p.id)), 2
          ) as porcentaje_evaluados
      FROM products p
      LEFT JOIN quality_products qp ON p.id = qp.product_id;
      ```
      
      ### **14. Ver el promedio de rating por encuesta**
      
      ```sql
      SELECT 
          p.name as encuesta,
          COUNT(qp.rating) as total_calificaciones,
          AVG(qp.rating) as rating_promedio,
          MIN(qp.rating) as rating_minimo,
          MAX(qp.rating) as rating_maximo,
          STDDEV(qp.rating) as desviacion_estandar
      FROM polls p
      LEFT JOIN quality_products qp ON p.id = qp.poll_id
      GROUP BY p.id, p.name
      ORDER BY rating_promedio DESC;
      ```
      
      ### **15. Calcular el promedio y total de beneficios por plan**
      
      ```sql
      SELECT 
          m.id as membership_id,
          m.name as membership_name,
          m.description as membership_description,
          COUNT(DISTINCT mb.benefit_id) as total_beneficios,
          COUNT(DISTINCT mb.benefit_id) * 1.0 as promedio_beneficios,
          ROUND(COUNT(DISTINCT mb.benefit_id) * 1.0 / COUNT(DISTINCT mb.period_id), 2) as promedio_beneficios_por_periodo
      FROM memberships m
      LEFT JOIN membershipbenefits mb ON m.id = mb.membership_id
      GROUP BY m.id, m.name, m.description
      ORDER BY total_beneficios DESC;
      ```
      
      ### **16. Obtener media y varianza de precios por empresa**
      
      ```sql
      SELECT 
          c.id as company_id,
          c.name as company_name,
          COUNT(cp.product_id) as total_productos,
          ROUND(AVG(cp.price), 2) as media_precio,
          ROUND(VAR_POP(cp.price), 2) as varianza_poblacional,
          ROUND(VAR_SAMP(cp.price), 2) as varianza_muestral,
          ROUND(STDDEV_POP(cp.price), 2) as desviacion_estandar_poblacional,
          ROUND(STDDEV_SAMP(cp.price), 2) as desviacion_estandar_muestral,
          MIN(cp.price) as precio_minimo,
          MAX(cp.price) as precio_maximo,
          ROUND(MAX(cp.price) - MIN(cp.price), 2) as rango_precios
      FROM companies c
      INNER JOIN companyproducts cp ON c.id = cp.company_id
      GROUP BY c.id, c.name
      HAVING COUNT(cp.product_id) > 0
      ORDER BY media_precio DESC;
      ```
      
      ### **17. Ver total de productos disponibles en la ciudad del cliente**
      
      ```sql
      SELECT 
          cust.id as customer_id,
          cust.name as customer_name,
          city.name as ciudad_cliente,
          state.name as estado_cliente,
          country.name as pais_cliente,
          COUNT(DISTINCT cp.product_id) as total_productos_disponibles,
          COUNT(DISTINCT comp.id) as total_empresas_en_ciudad,
          COUNT(DISTINCT cat.id) as total_categorias_disponibles
      FROM customers cust
      INNER JOIN citiesormunicipalities city ON cust.city_id = city.code
      INNER JOIN stateorregions state ON city.statereg_id = state.code
      INNER JOIN countries country ON state.country_id = country.isocode
      LEFT JOIN companies comp ON comp.city_id = city.code
      LEFT JOIN companyproducts cp ON comp.id = cp.company_id
      LEFT JOIN products p ON cp.product_id = p.id
      LEFT JOIN categories cat ON p.category_id = cat.id
      GROUP BY cust.id, cust.name, city.name, state.name, country.name
      ORDER BY total_productos_disponibles DESC;
      ```
      
      ### **18. Contar productos únicos por tipo de empresa**
      
      ```sql
      SELECT 
          ti.id as type_id,
          ti.description as tipo_empresa,
          COUNT(DISTINCT cp.product_id) as productos_unicos,
          COUNT(DISTINCT c.id) as total_empresas_tipo,
          COUNT(cp.product_id) as total_registros_productos,
          ROUND(AVG(cp.price), 2) as precio_promedio,
          MIN(cp.price) as precio_minimo,
          MAX(cp.price) as precio_maximo
      FROM typesofidentifications ti
      INNER JOIN companies c ON ti.id = c.type_id
      INNER JOIN companyproducts cp ON c.id = cp.company_id
      GROUP BY ti.id, ti.description
      ORDER BY productos_unicos DESC;
      ```
      
      ### **19. Ver total de clientes sin correo electrónico registrado**
      
      ```sql
      SELECT 
          COUNT(*) as total_clientes_sin_email
      FROM customers
      WHERE email IS NULL 
         OR email = '' 
         OR TRIM(email) = '';
         
      +--------------------------+
      | total_clientes_sin_email |
      +--------------------------+
      |                        0 |
      +--------------------------+
      ```
      
      ### **20. Empresa con más productos calificados**
      
      ```sql
      SELECT 
          c.id as company_id,
          c.name as company_name,
          COUNT(DISTINCT qp.product_id) as productos_calificados,
          COUNT(qp.product_id) as total_calificaciones,
          ROUND(AVG(qp.rating), 2) as rating_promedio,
          MIN(qp.rating) as rating_minimo,
          MAX(qp.rating) as rating_maximo,
          COUNT(DISTINCT qp.customer_id) as clientes_que_calificaron
      FROM companies c
      INNER JOIN companyproducts cp ON c.id = cp.company_id
      INNER JOIN quality_products qp ON cp.product_id = qp.product_id AND cp.company_id = qp.company_id
      GROUP BY c.id, c.name
      ORDER BY productos_calificados DESC, total_calificaciones DESC
      LIMIT 1;
      ```
      
      
      
      
      
      ##  **4. Procedimientos Almacenados**
      
      
      
      ### **1. Registrar una nueva calificación y actualizar el promedio**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE RegisterRating(
          IN p_product_id INTEGER,
          IN p_customer_id INTEGER,
          IN p_company_id VARCHAR(20),
          IN p_poll_id INTEGER,
          IN p_rating DOUBLE
      )
      BEGIN
          INSERT INTO quality_products (product_id, customer_id, poll_id, company_id, daterating, rating)
          VALUES (p_product_id, p_customer_id, p_poll_id, p_company_id, NOW(), p_rating);
          
          SELECT 
              p_product_id as product_id,
              p_company_id as company_id,
              ROUND(AVG(rating), 2) as promedio_actual,
              COUNT(*) as total_calificaciones
          FROM quality_products
          WHERE product_id = p_product_id AND company_id = p_company_id;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **2. Insertar empresa y asociar productos por defecto**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE InsertCompanyWithProducts(
          IN p_company_id VARCHAR(20),
          IN p_type_id INTEGER,
          IN p_name VARCHAR(80),
          IN p_category_id INTEGER,
          IN p_city_id VARCHAR(6),
          IN p_audience_id INTEGER,
          IN p_cellphone VARCHAR(15),
          IN p_email VARCHAR(80),
          IN p_default_price DOUBLE
      )
      BEGIN
          INSERT INTO companies (id, type_id, name, category_id, city_id, audience_id, cellphone, email)
          VALUES (p_company_id, p_type_id, p_name, p_category_id, p_city_id, p_audience_id, p_cellphone, p_email);
          
          INSERT INTO companyproducts (company_id, product_id, price, unitmeasure_id)
          SELECT 
              p_company_id,
              p.id,
              p_default_price,
              1
          FROM products p
          WHERE p.category_id = p_category_id;
          
          SELECT 
              'Empresa creada exitosamente' as mensaje,
              p_company_id as company_id,
              p_name as company_name,
              COUNT(cp.product_id) as productos_asociados
          FROM companies c
          LEFT JOIN companyproducts cp ON c.id = cp.company_id
          WHERE c.id = p_company_id
          GROUP BY c.id, c.name;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **3. Añadir producto favorito validando duplicados**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE AddFavoriteProduct(
          IN p_customer_id INTEGER,
          IN p_company_id VARCHAR(20)
      )
      BEGIN
          DECLARE v_exists INTEGER DEFAULT 0;
          DECLARE v_favorite_id INTEGER;
          
          SELECT COUNT(*) INTO v_exists
          FROM favorites
          WHERE customer_id = p_customer_id AND company_id = p_company_id;
          
          IF v_exists > 0 THEN
              SELECT 
                  'La empresa ya está en favoritos' as mensaje,
                  p_customer_id as customer_id,
                  p_company_id as company_id,
                  'DUPLICADO' as estado;
          ELSE
              INSERT INTO favorites (customer_id, company_id)
              VALUES (p_customer_id, p_company_id);
              
              SET v_favorite_id = LAST_INSERT_ID();
              
              INSERT INTO details_favorites (favorite_id, product_id)
              SELECT v_favorite_id, cp.product_id
              FROM companyproducts cp
              WHERE cp.company_id = p_company_id;
              
              SELECT 
                  'Favorito añadido exitosamente' as mensaje,
                  p_customer_id as customer_id,
                  p_company_id as company_id,
                  v_favorite_id as favorite_id,
                  COUNT(df.product_id) as productos_favoritos,
                  'NUEVO' as estado
              FROM details_favorites df
              WHERE df.favorite_id = v_favorite_id;
          END IF;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **4. Generar resumen mensual de calificaciones por empresa**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE MonthlyRatingsSummary(
          IN p_year INTEGER,
          IN p_month INTEGER
      )
      BEGIN
          SELECT 
              c.id as company_id,
              c.name as company_name,
              ti.description as tipo_empresa,
              city.name as ciudad,
              COUNT(qp.rating) as total_calificaciones,
              ROUND(AVG(qp.rating), 2) as promedio_calificaciones,
              MIN(qp.rating) as calificacion_minima,
              MAX(qp.rating) as calificacion_maxima,
              COUNT(DISTINCT qp.product_id) as productos_calificados,
              COUNT(DISTINCT qp.customer_id) as clientes_que_calificaron,
              CONCAT(p_year, '-', LPAD(p_month, 2, '0')) as periodo
          FROM companies c
          INNER JOIN companyproducts cp ON c.id = cp.company_id
          INNER JOIN quality_products qp ON cp.product_id = qp.product_id AND cp.company_id = qp.company_id
          LEFT JOIN typesofidentifications ti ON c.type_id = ti.id
          LEFT JOIN citiesormunicipalities city ON c.city_id = city.code
          WHERE YEAR(qp.daterating) = p_year 
            AND MONTH(qp.daterating) = p_month
          GROUP BY c.id, c.name, ti.description, city.name
          ORDER BY promedio_calificaciones DESC, total_calificaciones DESC;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **5. Calcular beneficios activos por membresía**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE ActiveBenefitsByMembership()
      BEGIN
          SELECT 
              m.id as membership_id,
              m.name as membership_name,
              m.description as membership_description,
              COUNT(DISTINCT mb.benefit_id) as total_beneficios_activos,
              COUNT(DISTINCT mb.period_id) as periodos_activos,
              COUNT(DISTINCT mb.audience_id) as audiencias_cubiertas,
              GROUP_CONCAT(DISTINCT b.description ORDER BY b.description SEPARATOR ', ') as lista_beneficios
          FROM memberships m
          INNER JOIN membershipbenefits mb ON m.id = mb.membership_id
          INNER JOIN benefits b ON mb.benefit_id = b.id
          GROUP BY m.id, m.name, m.description
          ORDER BY total_beneficios_activos DESC;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **6. Eliminar productos huérfanos**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE EliminarProductosSinCalificacionNiEmpresa()
      BEGIN
          DECLARE EXIT HANDLER FOR SQLEXCEPTION
          BEGIN
              ROLLBACK;
              RESIGNAL;
          END;
      
          START TRANSACTION;
      
          DELETE p FROM products p
          WHERE p.id NOT IN (
              SELECT DISTINCT qp.product_id 
              FROM quality_products qp 
              WHERE qp.product_id IS NOT NULL
          )
          AND p.id NOT IN (
              SELECT DISTINCT cp.product_id 
              FROM companyproducts cp 
              WHERE cp.product_id IS NOT NULL
          );
      
          SELECT ROW_COUNT() as productos_eliminados;
      
          COMMIT;
      END//
      
      DELIMITER ;
      ```
      
      ### **7. Actualizar precios de productos por categoría**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE ActualizarPreciosPorCategoria(
          IN categoria_id INT,
          IN porcentaje DOUBLE
      )
      BEGIN
          UPDATE products 
          SET price = price * (1 + (porcentaje / 100))
          WHERE category_id = categoria_id;
          
          SELECT ROW_COUNT() as productos_actualizados;
      END//
      
      DELIMITER ;
      ```
      
      ### **8. Validar inconsistencia entre `rates` y `quality_products`**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE AuditarInconsistenciasRatesQuality()
      BEGIN
          SELECT 
              'Productos con rates sin quality_products' as tipo_inconsistencia,
              r.customer_id,
              r.company_id,
              r.poll_id,
              r.rating,
              r.daterating,
              'N/A' as product_id_quality,
              'N/A' as rating_quality
          FROM rates r
          LEFT JOIN quality_products qp ON r.customer_id = qp.customer_id 
                                        AND r.company_id = qp.company_id 
                                        AND r.poll_id = qp.poll_id
          WHERE qp.customer_id IS NULL
          
          UNION ALL
          
          SELECT 
              'Productos con quality_products sin rates' as tipo_inconsistencia,
              CAST(qp.customer_id AS CHAR) as customer_id,
              qp.company_id,
              CAST(qp.poll_id AS CHAR) as poll_id,
              CAST(qp.rating AS CHAR) as rating,
              CAST(qp.daterating AS CHAR) as daterating,
              CAST(qp.product_id AS CHAR) as product_id_quality,
              CAST(qp.rating AS CHAR) as rating_quality
          FROM quality_products qp
          LEFT JOIN rates r ON qp.customer_id = r.customer_id 
                            AND qp.company_id = r.company_id 
                            AND qp.poll_id = r.poll_id
          WHERE r.customer_id IS NULL
          
          UNION ALL
          
          SELECT 
              'Diferencias en ratings' as tipo_inconsistencia,
              CAST(r.customer_id AS CHAR) as customer_id,
              r.company_id,
              CAST(r.poll_id AS CHAR) as poll_id,
              CONCAT('rates:', r.rating, ' vs quality:', qp.rating) as rating,
              CAST(r.daterating AS CHAR) as daterating,
              CAST(qp.product_id AS CHAR) as product_id_quality,
              CAST(qp.rating AS CHAR) as rating_quality
          FROM rates r
          INNER JOIN quality_products qp ON r.customer_id = qp.customer_id 
                                         AND r.company_id = qp.company_id 
                                         AND r.poll_id = qp.poll_id
          WHERE r.rating != qp.rating
          
          ORDER BY tipo_inconsistencia, customer_id;
          
          SELECT 
              'RESUMEN DE INCONSISTENCIAS' as resumen,
              (SELECT COUNT(*) FROM rates r
               LEFT JOIN quality_products qp ON r.customer_id = qp.customer_id 
                                              AND r.company_id = qp.company_id 
                                              AND r.poll_id = qp.poll_id
               WHERE qp.customer_id IS NULL) as rates_sin_quality,
              
              (SELECT COUNT(*) FROM quality_products qp
               LEFT JOIN rates r ON qp.customer_id = r.customer_id 
                                  AND qp.company_id = r.company_id 
                                  AND qp.poll_id = r.poll_id
               WHERE r.customer_id IS NULL) as quality_sin_rates,
              
              (SELECT COUNT(*) FROM rates r
               INNER JOIN quality_products qp ON r.customer_id = qp.customer_id 
                                              AND r.company_id = qp.company_id 
                                              AND r.poll_id = qp.poll_id
               WHERE r.rating != qp.rating) as diferencias_rating;
      END//
      
      DELIMITER ;
      ```
      
      ### **9. Asignar beneficios a nuevas audiencias**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE AsignarBeneficiosANuevasAudiencias(
          IN audience_id INT,
          IN benefit_id INT
      )
      BEGIN
          DECLARE audience_existe INT DEFAULT 0;
          DECLARE benefit_existe INT DEFAULT 0;
          DECLARE ya_asignado INT DEFAULT 0;
          
          SELECT COUNT(*) INTO audience_existe 
          FROM audiences 
          WHERE id = audience_id;
          
          SELECT COUNT(*) INTO benefit_existe 
          FROM benefits 
          WHERE id = benefit_id;
          
          SELECT COUNT(*) INTO ya_asignado 
          FROM audiencebenefits 
          WHERE audience_id = audience_id AND benefit_id = benefit_id;
          
          IF audience_existe = 0 THEN
              SELECT 'Error: La audiencia no existe' as resultado;
          ELSEIF benefit_existe = 0 THEN
              SELECT 'Error: El beneficio no existe' as resultado;
          ELSEIF ya_asignado > 0 THEN
              SELECT 'Error: El beneficio ya está asignado a esta audiencia' as resultado;
          ELSE
              INSERT INTO audiencebenefits (audience_id, benefit_id) 
              VALUES (audience_id, benefit_id);
              
              SELECT 
                  'Beneficio asignado exitosamente' as resultado,
                  a.description as audiencia,
                  b.description as beneficio
              FROM audiences a, benefits b
              WHERE a.id = audience_id AND b.id = benefit_id;
          END IF;
      END//
      
      DELIMITER ;
      ```
      
      ### **10. Activar planes de membresía vencidos con pago confirmado**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE ActivarMembresiasVencidas()
      BEGIN
          DECLARE membresias_activadas INT DEFAULT 0;
          
          UPDATE memberships m
          INNER JOIN membershipperiods mp ON m.id = mp.membership_id
          SET m.description = CONCAT(m.description, ' - REACTIVADA')
          WHERE mp.price > 0 
          AND m.id IN (
              SELECT DISTINCT membership_id 
              FROM membershipperiods 
              WHERE price > 0
          );
          
          SET membresias_activadas = ROW_COUNT();
          
          SELECT 
              membresias_activadas as membresias_reactivadas,
              NOW() as fecha_activacion,
              'Membresías vencidas con pago confirmado han sido reactivadas' as mensaje;
              
          SELECT 
              m.id as membership_id,
              m.name as nombre_membresia,
              m.description as descripcion,
              mp.price as precio_pagado,
              p.name as periodo
          FROM memberships m
          INNER JOIN membershipperiods mp ON m.id = mp.membership_id
          INNER JOIN periods p ON mp.period_id = p.id
          WHERE m.description LIKE '%REACTIVADA%'
          AND mp.price > 0;
      END//
      
      DELIMITER ;
      ```
      
      ### **11. Listar productos favoritos del cliente con su calificación**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE ObtenerProductosFavoritos(
          IN customer_id INT
      )
      BEGIN
          IF NOT EXISTS (SELECT 1 FROM customers WHERE id = customer_id) THEN
              SELECT 'Error: El cliente no existe' as mensaje;
          ELSE
              SELECT 
                  f.id as favorito_id,
                  f.customer_id,
                  c.name as nombre_cliente,
                  p.id as product_id,
                  p.name as nombre_producto,
                  p.detail as detalle_producto,
                  p.price as precio,
                  cat.description as categoria,
                  comp.name as empresa,
                  COALESCE(ROUND(AVG(qp.rating), 2), 0) as promedio_rating,
                  COUNT(qp.rating) as total_calificaciones
              FROM favorites f
              INNER JOIN customers c ON f.customer_id = c.id
              INNER JOIN products p ON f.product_id = p.id
              INNER JOIN categories cat ON p.category_id = cat.id
              LEFT JOIN companyproducts cp ON p.id = cp.product_id
              LEFT JOIN companies comp ON cp.company_id = comp.id
              LEFT JOIN quality_products qp ON p.id = qp.product_id
              WHERE f.customer_id = customer_id
              GROUP BY f.id, f.customer_id, c.name, p.id, p.name, p.detail, 
                       p.price, cat.description, comp.name
              ORDER BY promedio_rating DESC, p.name;
              
              SELECT 
                  customer_id as cliente_id,
                  COUNT(*) as total_favoritos,
                  ROUND(AVG(sub.promedio_rating), 2) as promedio_general_favoritos
              FROM (
                  SELECT 
                      f.customer_id,
                      COALESCE(AVG(qp.rating), 0) as promedio_rating
                  FROM favorites f
                  INNER JOIN products p ON f.product_id = p.id
                  LEFT JOIN quality_products qp ON p.id = qp.product_id
                  WHERE f.customer_id = customer_id
                  GROUP BY f.id, f.customer_id
              ) sub
              GROUP BY customer_id;
          END IF;
      END//
      
      DELIMITER ;
      ```
      
      ### **12. Registrar encuesta y sus preguntas asociadas**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE RegistrarEncuestaConPreguntas(
          IN p_nombre_encuesta VARCHAR(80),
          IN p_descripcion TEXT,
          IN p_categoria_poll_id INT,
          IN p_activa BOOLEAN
      )
      BEGIN
          DECLARE nueva_encuesta_id INT DEFAULT 0;
          DECLARE categoria_existe INT DEFAULT 0;
          DECLARE EXIT HANDLER FOR SQLEXCEPTION
          BEGIN
              ROLLBACK;
              RESIGNAL;
          END;
      
          START TRANSACTION;
          
          SELECT COUNT(*) INTO categoria_existe 
          FROM categories_polls 
          WHERE id = p_categoria_poll_id;
          
          IF categoria_existe = 0 THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = 'Error: La categoría de encuesta no existe';
          END IF;
          
          INSERT INTO polls (name, description, isactive, categorypoll_id)
          VALUES (p_nombre_encuesta, p_descripcion, p_activa, p_categoria_poll_id);
          
          SET nueva_encuesta_id = LAST_INSERT_ID();
          
          SELECT 
              nueva_encuesta_id as encuesta_id,
              p_nombre_encuesta as nombre,
              p_descripcion as descripcion,
              cp.name as categoria,
              p_activa as activa,
              'Encuesta registrada exitosamente' as mensaje
          FROM categories_polls cp
          WHERE cp.id = p_categoria_poll_id;
          
          COMMIT;
      END//
      
      DELIMITER ;
      
      ```
      
      ### **13. Eliminar favoritos antiguos sin calificaciones**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE EliminarFavoritosAntiguosSinCalificar()
      BEGIN
          DECLARE favoritos_eliminados INT DEFAULT 0;
          DECLARE fecha_limite DATE;
          
          SET fecha_limite = DATE_SUB(CURDATE(), INTERVAL 1 YEAR);
          
          DELETE f FROM favorites f
          INNER JOIN details_favorites df ON f.id = df.favorite_id
          WHERE f.product_id NOT IN (
              SELECT DISTINCT qp.product_id 
              FROM quality_products qp 
              WHERE qp.customer_id = f.customer_id
              AND qp.product_id IS NOT NULL
              AND qp.daterating >= fecha_limite
          )
          AND f.id IN (
              SELECT df2.favorite_id 
              FROM details_favorites df2 
              WHERE df2.favorite_id IS NOT NULL
          );
          
          SET favoritos_eliminados = ROW_COUNT();
          
          SELECT 
              favoritos_eliminados as favoritos_eliminados,
              fecha_limite as fecha_limite_calificacion,
              'Favoritos antiguos sin calificación eliminados' as mensaje;
              
          SELECT 
              COUNT(*) as favoritos_restantes,
              COUNT(DISTINCT f.customer_id) as clientes_con_favoritos,
              COUNT(DISTINCT f.product_id) as productos_favoritos_unicos
          FROM favorites f;
      END//
      
      DELIMITER ;
      
      ```
      
      ### **14. Asociar beneficios automáticamente por audiencia**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE AsociarBeneficiosPorAudiencia()
      BEGIN
          DECLARE beneficios_asignados INT DEFAULT 0;
          DECLARE audiencias_procesadas INT DEFAULT 0;
          
          INSERT INTO audiencebenefits (audience_id, benefit_id)
          SELECT DISTINCT a.id, b.id
          FROM audiences a
          CROSS JOIN benefits b
          WHERE NOT EXISTS (
              SELECT 1 FROM audiencebenefits ab 
              WHERE ab.audience_id = a.id 
              AND ab.benefit_id = b.id
          );
          
          SET beneficios_asignados = ROW_COUNT();
          
          SELECT COUNT(DISTINCT audience_id) INTO audiencias_procesadas
          FROM audiencebenefits;
          
          SELECT 
              beneficios_asignados as nuevos_beneficios_asignados,
              audiencias_procesadas as audiencias_con_beneficios,
              'Beneficios asociados automáticamente' as mensaje;
              
          SELECT 
              a.id as audiencia_id,
              a.description as audiencia,
              COUNT(ab.benefit_id) as total_beneficios
          FROM audiences a
          LEFT JOIN audiencebenefits ab ON a.id = ab.audience_id
          GROUP BY a.id, a.description
          ORDER BY total_beneficios DESC;
      END//
      
      DELIMITER ;
      
      ```
      
      ### **15. Historial de cambios de precio**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE GenerarHistorialCambiosPrecios(
          IN fecha_inicio DATE,
          IN fecha_fin DATE,
          IN producto_id INT DEFAULT NULL
      )
      BEGIN
          CREATE TEMPORARY TABLE temp_price_history (
              producto_id INT,
              nombre_producto VARCHAR(60),
              categoria VARCHAR(60),
              precio_anterior DOUBLE,
              precio_actual DOUBLE,
              cambio_absoluto DOUBLE,
              cambio_porcentual DOUBLE,
              fecha_cambio DATETIME,
              tipo_cambio VARCHAR(20)
          );
          
          INSERT INTO temp_price_history
          SELECT 
              p.id as producto_id,
              p.name as nombre_producto,
              c.description as categoria,
              p.price as precio_anterior,
              cp.price as precio_actual,
              (cp.price - p.price) as cambio_absoluto,
              CASE 
                  WHEN p.price > 0 THEN ROUND(((cp.price - p.price) / p.price) * 100, 2)
                  ELSE 0 
              END as cambio_porcentual,
              NOW() as fecha_cambio,
              CASE 
                  WHEN cp.price > p.price THEN 'AUMENTO'
                  WHEN cp.price < p.price THEN 'DESCUENTO'
                  ELSE 'SIN CAMBIO'
              END as tipo_cambio
          FROM products p
          INNER JOIN categories c ON p.category_id = c.id
          INNER JOIN companyproducts cp ON p.id = cp.product_id
          WHERE (producto_id IS NULL OR p.id = producto_id)
          AND p.price != cp.price
          AND (fecha_inicio IS NULL OR DATE(NOW()) >= fecha_inicio)
          AND (fecha_fin IS NULL OR DATE(NOW()) <= fecha_fin);
          
          INSERT INTO temp_price_history
          SELECT 
              m.id as producto_id,
              m.name as nombre_producto,
              'MEMBRESIA' as categoria,
              0 as precio_anterior,
              mp.price as precio_actual,
              mp.price as cambio_absoluto,
              100 as cambio_porcentual,
              NOW() as fecha_cambio,
              'PRECIO_MEMBRESIA' as tipo_cambio
          FROM memberships m
          INNER JOIN membershipperiods mp ON m.id = mp.membership_id
          WHERE mp.price > 0
          AND (producto_id IS NULL OR m.id = producto_id)
          AND (fecha_inicio IS NULL OR DATE(NOW()) >= fecha_inicio)
          AND (fecha_fin IS NULL OR DATE(NOW()) <= fecha_fin);
          
          SELECT 
              producto_id,
              nombre_producto,
              categoria,
              precio_anterior,
              precio_actual,
              cambio_absoluto,
              CONCAT(cambio_porcentual, '%') as cambio_porcentual,
              fecha_cambio,
              tipo_cambio
          FROM temp_price_history
          ORDER BY fecha_cambio DESC, cambio_porcentual DESC;
          
          SELECT 
              COUNT(*) as total_cambios,
              SUM(CASE WHEN tipo_cambio = 'AUMENTO' THEN 1 ELSE 0 END) as aumentos,
              SUM(CASE WHEN tipo_cambio = 'DESCUENTO' THEN 1 ELSE 0 END) as descuentos,
              ROUND(AVG(cambio_absoluto), 2) as promedio_cambio_absoluto,
              ROUND(AVG(cambio_porcentual), 2) as promedio_cambio_porcentual,
              MAX(cambio_porcentual) as mayor_aumento_porcentual,
              MIN(cambio_porcentual) as mayor_descuento_porcentual
          FROM temp_price_history;
          
          DROP TEMPORARY TABLE temp_price_history;
      END//
      
      DELIMITER ;
      ```
      
      ### **16. Registrar encuesta activa automáticamente**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE RegisterNewActivePoll(
          IN p_name VARCHAR(80),
          IN p_description TEXT,
          IN p_categorypoll_id INTEGER
      )
      BEGIN
          DECLARE v_poll_id INTEGER;
          DECLARE v_error_message VARCHAR(255);
          
          DECLARE EXIT HANDLER FOR SQLEXCEPTION
          BEGIN
              ROLLBACK;
              GET DIAGNOSTICS CONDITION 1
                  v_error_message = MESSAGE_TEXT;
              SELECT CONCAT('Error al crear la encuesta: ', v_error_message) AS error_message;
          END;
          
          START TRANSACTION;
          
          IF p_name IS NULL OR TRIM(p_name) = '' THEN
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'El nombre de la encuesta es obligatorio';
          END IF;
          
          IF p_categorypoll_id IS NOT NULL THEN
              IF NOT EXISTS (SELECT 1 FROM categories_polls WHERE id = p_categorypoll_id) THEN
                  SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La categoría de encuesta especificada no existe';
              END IF;
          END IF;
          
          INSERT INTO polls (
              name, 
              description, 
              isactive, 
              categorypoll_id
          ) VALUES (
              TRIM(p_name),
              COALESCE(p_description, ''),
              TRUE,  -- Automáticamente activa
              p_categorypoll_id
          );
          
          SET v_poll_id = LAST_INSERT_ID();
          
          COMMIT;
          
          SELECT 
              v_poll_id as poll_id,
              p_name as poll_name,
              p_description as poll_description,
              TRUE as is_active,
              p_categorypoll_id as category_id,
              cp.name as category_name,
              NOW() as created_at,
              'Encuesta creada exitosamente' as message
          FROM polls p
          LEFT JOIN categories_polls cp ON p.categorypoll_id = cp.id
          WHERE p.id = v_poll_id;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **17. Actualizar unidad de medida de productos sin afectar ventas**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE UpdateProductUnitMeasure(
          IN p_product_id INTEGER,
          IN p_new_unit_measure_id INTEGER
      )
      BEGIN
          DECLARE v_has_sales INTEGER DEFAULT 0;
          DECLARE v_product_exists INTEGER DEFAULT 0;
          DECLARE v_unit_exists INTEGER DEFAULT 0;
          
          SELECT COUNT(*) INTO v_product_exists
          FROM products 
          WHERE id = p_product_id;
          
          IF v_product_exists = 0 THEN
              SELECT 'Error: El producto no existe' AS message;
              LEAVE sp;
          END IF;
          
          SELECT COUNT(*) INTO v_unit_exists
          FROM unitofmeasure 
          WHERE id = p_new_unit_measure_id;
          
          IF v_unit_exists = 0 THEN
              SELECT 'Error: La unidad de medida no existe' AS message;
              LEAVE sp;
          END IF;
          
          SELECT COUNT(*) INTO v_has_sales
          FROM companyproducts 
          WHERE product_id = p_product_id;
          
          IF v_has_sales = 0 THEN
              CREATE TABLE IF NOT EXISTS product_default_units (
                  product_id INTEGER PRIMARY KEY,
                  unit_measure_id INTEGER,
                  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                  FOREIGN KEY (product_id) REFERENCES products(id),
                  FOREIGN KEY (unit_measure_id) REFERENCES unitofmeasure(id)
              );
              
              INSERT INTO product_default_units (product_id, unit_measure_id)
              VALUES (p_product_id, p_new_unit_measure_id)
              ON DUPLICATE KEY UPDATE 
                  unit_measure_id = p_new_unit_measure_id,
                  updated_at = CURRENT_TIMESTAMP;
              
              SELECT 
                  p_product_id as product_id,
                  'Unidad de medida actualizada exitosamente' as message,
                  'Sin ventas previas' as status
              FROM DUAL;
          ELSE
              CREATE TABLE IF NOT EXISTS product_default_units (
                  product_id INTEGER PRIMARY KEY,
                  unit_measure_id INTEGER,
                  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                  FOREIGN KEY (product_id) REFERENCES products(id),
                  FOREIGN KEY (unit_measure_id) REFERENCES unitofmeasure(id)
              );
              
              INSERT INTO product_default_units (product_id, unit_measure_id)
              VALUES (p_product_id, p_new_unit_measure_id)
              ON DUPLICATE KEY UPDATE 
                  unit_measure_id = p_new_unit_measure_id,
                  updated_at = CURRENT_TIMESTAMP;
              
              SELECT 
                  p_product_id as product_id,
                  'Configuración guardada para nuevas ventas' as message,
                  CONCAT('Ventas existentes: ', v_has_sales) as status
              FROM DUAL;
          END IF;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **18. Recalcular promedios de calidad semanalmente**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE RecalculateQualityAverages()
      BEGIN
          DECLARE v_total_products INTEGER DEFAULT 0;
          DECLARE v_products_with_ratings INTEGER DEFAULT 0;
          DECLARE v_global_average DECIMAL(10,2) DEFAULT 0.00;
          DECLARE v_execution_time TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
          
          CREATE TABLE IF NOT EXISTS quality_statistics (
              id INTEGER AUTO_INCREMENT PRIMARY KEY,
              product_id INTEGER,
              product_name VARCHAR(60),
              average_rating DECIMAL(10,2),
              total_ratings INTEGER,
              min_rating DECIMAL(10,2),
              max_rating DECIMAL(10,2),
              last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
              FOREIGN KEY (product_id) REFERENCES products(id)
          );
          
          CREATE TABLE IF NOT EXISTS global_quality_summary (
              id INTEGER AUTO_INCREMENT PRIMARY KEY,
              total_products INTEGER,
              products_with_ratings INTEGER,
              global_average DECIMAL(10,2),
              execution_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
          );
          
          DELETE FROM quality_statistics;
          
          INSERT INTO quality_statistics (
              product_id, 
              product_name, 
              average_rating, 
              total_ratings, 
              min_rating, 
              max_rating,
              last_updated
          )
          SELECT 
              p.id,
              p.name,
              AVG(qp.rating) as average_rating,
              COUNT(qp.rating) as total_ratings,
              MIN(qp.rating) as min_rating,
              MAX(qp.rating) as max_rating,
              v_execution_time
          FROM products p
          INNER JOIN quality_products qp ON p.id = qp.product_id
          WHERE qp.rating IS NOT NULL
          GROUP BY p.id, p.name;
          
          SELECT COUNT(*) INTO v_total_products FROM products;
          SELECT COUNT(DISTINCT product_id) INTO v_products_with_ratings FROM quality_products WHERE rating IS NOT NULL;
          SELECT AVG(rating) INTO v_global_average FROM quality_products WHERE rating IS NOT NULL;
          
          INSERT INTO global_quality_summary (
              total_products,
              products_with_ratings,
              global_average,
              execution_date
          ) VALUES (
              v_total_products,
              v_products_with_ratings,
              v_global_average,
              v_execution_time
          );
          
          SELECT 
              'Recálculo completado exitosamente' as message,
              v_total_products as total_products,
              v_products_with_ratings as products_with_ratings,
              v_global_average as global_average,
              v_execution_time as execution_time;
          
          SELECT 
              'Top 5 productos con mejor calidad' as info,
              product_name,
              average_rating,
              total_ratings
          FROM quality_statistics
          ORDER BY average_rating DESC
          LIMIT 5;
          
          SELECT 
              'Productos que necesitan atención' as info,
              product_name,
              average_rating,
              total_ratings
          FROM quality_statistics
          WHERE average_rating < 3.0
          ORDER BY average_rating ASC;
          
      END //
      
      DELIMITER ;
      ```
      
      ### **19. Validar claves foráneas entre calificaciones y encuestas**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE ValidateCrossForeignKeys()
      BEGIN
          DECLARE v_validation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP;
          DECLARE v_total_errors INTEGER DEFAULT 0;
          
          CREATE TABLE IF NOT EXISTS validation_errors (
              id INTEGER AUTO_INCREMENT PRIMARY KEY,
              table_name VARCHAR(50),
              error_type VARCHAR(100),
              record_id VARCHAR(50),
              error_description TEXT,
              validation_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
          );
          
          DELETE FROM validation_errors WHERE validation_date < DATE_SUB(NOW(), INTERVAL 1 HOUR);
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'quality_products' as table_name,
              'Producto inexistente' as error_type,
              CONCAT('product_id: ', qp.product_id) as record_id,
              CONCAT('El producto con ID ', qp.product_id, ' no existe en la tabla products') as error_description,
              v_validation_date
          FROM quality_products qp
          LEFT JOIN products p ON qp.product_id = p.id
          WHERE p.id IS NULL AND qp.product_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'quality_products' as table_name,
              'Cliente inexistente' as error_type,
              CONCAT('customer_id: ', qp.customer_id) as record_id,
              CONCAT('El cliente con ID ', qp.customer_id, ' no existe en la tabla customers') as error_description,
              v_validation_date
          FROM quality_products qp
          LEFT JOIN customers c ON qp.customer_id = c.id
          WHERE c.id IS NULL AND qp.customer_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'quality_products' as table_name,
              'Encuesta inexistente' as error_type,
              CONCAT('poll_id: ', qp.poll_id) as record_id,
              CONCAT('La encuesta con ID ', qp.poll_id, ' no existe en la tabla polls') as error_description,
              v_validation_date
          FROM quality_products qp
          LEFT JOIN polls p ON qp.poll_id = p.id
          WHERE p.id IS NULL AND qp.poll_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'quality_products' as table_name,
              'Empresa inexistente' as error_type,
              CONCAT('company_id: ', qp.company_id) as record_id,
              CONCAT('La empresa con ID ', qp.company_id, ' no existe en la tabla companies') as error_description,
              v_validation_date
          FROM quality_products qp
          LEFT JOIN companies c ON qp.company_id = c.id
          WHERE c.id IS NULL AND qp.company_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'rates' as table_name,
              'Cliente inexistente' as error_type,
              CONCAT('customer_id: ', r.customer_id) as record_id,
              CONCAT('El cliente con ID ', r.customer_id, ' no existe en la tabla customers') as error_description,
              v_validation_date
          FROM rates r
          LEFT JOIN customers c ON r.customer_id = c.id
          WHERE c.id IS NULL AND r.customer_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'rates' as table_name,
              'Empresa inexistente' as error_type,
              CONCAT('company_id: ', r.company_id) as record_id,
              CONCAT('La empresa con ID ', r.company_id, ' no existe en la tabla companies') as error_description,
              v_validation_date
          FROM rates r
          LEFT JOIN companies c ON r.company_id = c.id
          WHERE c.id IS NULL AND r.company_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'rates' as table_name,
              'Encuesta inexistente' as error_type,
              CONCAT('poll_id: ', r.poll_id) as record_id,
              CONCAT('La encuesta con ID ', r.poll_id, ' no existe en la tabla polls') as error_description,
              v_validation_date
          FROM rates r
          LEFT JOIN polls p ON r.poll_id = p.id
          WHERE p.id IS NULL AND r.poll_id IS NOT NULL;
          
          INSERT INTO validation_errors (table_name, error_type, record_id, error_description, validation_date)
          SELECT 
              'quality_products' as table_name,
              'Encuesta inactiva' as error_type,
              CONCAT('poll_id: ', qp.poll_id) as record_id,
              CONCAT('La encuesta "', p.name, '" está inactiva pero tiene calificaciones') as error_description,
              v_validation_date
          FROM quality_products qp
          INNER JOIN polls p ON qp.poll_id = p.id
          WHERE p.isactive = FALSE;
          
          SELECT COUNT(*) INTO v_total_errors 
          FROM validation_errors 
          WHERE validation_date = v_validation_date;
          
          SELECT 
              'Validación completada' as message,
              v_total_errors as total_errors,
              v_validation_date as validation_date;
          
          SELECT 
              table_name,
              error_type,
              COUNT(*) as error_count
          FROM validation_errors
          WHERE validation_date = v_validation_date
          GROUP BY table_name, error_type
          ORDER BY error_count DESC;
          
          IF v_total_errors <= 20 THEN
              SELECT 
                  table_name,
                  error_type,
                  record_id,
                  error_description
              FROM validation_errors
              WHERE validation_date = v_validation_date
              ORDER BY table_name, error_type;
          END IF;
          
      END //
      
      DELIMITER ;
      
      DELIMITER //
      
      CREATE PROCEDURE GetValidationHistory()
      BEGIN
          SELECT 
              DATE(validation_date) as validation_day,
              COUNT(*) as total_errors,
              COUNT(DISTINCT table_name) as tables_affected
          FROM validation_errors
          GROUP BY DATE(validation_date)
          ORDER BY validation_day DESC
          LIMIT 10;
      END //
      
      DELIMITER ;
      ```
      
      ### **20. Generar el top 10 de productos más calificados por ciudad**
      
      ```sql
      DELIMITER //
      
      CREATE PROCEDURE GetTop10ProductsByCity(IN city_code VARCHAR(6))
      BEGIN
          DECLARE city_exists INT DEFAULT 0;
          
          SELECT COUNT(*) INTO city_exists 
          FROM citiesormunicipalities 
          WHERE code = city_code;
          
          IF city_exists = 0 THEN
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'La ciudad especificada no existe';
          ELSE
              SELECT 
                  p.id AS product_id,
                  p.name AS product_name,
                  p.detail AS product_detail,
                  p.price AS product_price,
                  c.description AS category_name,
                  cm.name AS city_name,
                  COUNT(qp.rating) AS total_ratings,
                  AVG(qp.rating) AS average_rating,
                  ROUND(AVG(qp.rating), 2) AS rounded_average_rating,
                  MAX(qp.rating) AS max_rating,
                  MIN(qp.rating) AS min_rating
              FROM products p
              INNER JOIN categories c ON p.category_id = c.id
              INNER JOIN quality_products qp ON p.id = qp.product_id
              INNER JOIN companies comp ON qp.company_id = comp.id
              INNER JOIN citiesormunicipalities cm ON comp.city_id = cm.code
              WHERE cm.code = city_code
              GROUP BY 
                  p.id, 
                  p.name, 
                  p.detail, 
                  p.price, 
                  c.description, 
                  cm.name
              HAVING COUNT(qp.rating) > 0
              ORDER BY 
                  AVG(qp.rating) DESC,
                  COUNT(qp.rating) DESC,
                  p.name ASC
              LIMIT 10;
          END IF;
      END//
      
      DELIMITER ;
      
      ```
      
      
      
      ## **5. Triggers**
      
      ### **1. Actualizar la fecha de modificación de un producto**
      
      ```sql
      //ALTER TABLE products 
      ADD COLUMN created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      ADD COLUMN updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;//
      
      DELIMITER //
      
      CREATE TRIGGER tr_products_update_timestamp
          BEFORE UPDATE ON products
          FOR EACH ROW
      BEGIN
          SET NEW.updated_at = CURRENT_TIMESTAMP;
          
          SET NEW.created_at = OLD.created_at;
          
      END//
      
      DELIMITER ;
      ```
      
      ### **2. Registrar log cuando un cliente califica un producto**
      
      ```sql
      //CREATE TABLE IF NOT EXISTS product_rating_log (
          id INT AUTO_INCREMENT PRIMARY KEY,
          action_type ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
          customer_id INT NOT NULL,
          product_id INT NOT NULL,
          company_id VARCHAR(20) NOT NULL,
          poll_id INT NOT NULL,
          old_rating DOUBLE NULL,
          new_rating DOUBLE NULL,
          rating_date DATETIME NULL,
          customer_name VARCHAR(60) NULL,
          customer_email VARCHAR(100) NULL,
          product_name VARCHAR(60) NULL,
          company_name VARCHAR(80) NULL,
          action_timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
          user_session VARCHAR(100) DEFAULT USER(),
          ip_address VARCHAR(45) NULL,
          user_agent TEXT NULL,
          INDEX idx_customer_id (customer_id),
          INDEX idx_product_id (product_id),
          INDEX idx_company_id (company_id),
          INDEX idx_action_timestamp (action_timestamp),
          INDEX idx_action_type (action_type)
      );//
      
      DELIMITER //
      
      CREATE TRIGGER tr_product_rating_insert_log
          AFTER INSERT ON quality_products
          FOR EACH ROW
      BEGIN
          DECLARE v_customer_name VARCHAR(60);
          DECLARE v_customer_email VARCHAR(100);
          DECLARE v_product_name VARCHAR(60);
          DECLARE v_company_name VARCHAR(80);
          
          SELECT name, email 
          INTO v_customer_name, v_customer_email
          FROM customers 
          WHERE id = NEW.customer_id;
          
          SELECT name 
          INTO v_product_name
          FROM products 
          WHERE id = NEW.product_id;
          
          SELECT name 
          INTO v_company_name
          FROM companies 
          WHERE id = NEW.company_id;
          
          INSERT INTO product_rating_log (
              action_type,
              customer_id,
              product_id,
              company_id,
              poll_id,
              old_rating,
              new_rating,
              rating_date,
              customer_name,
              customer_email,
              product_name,
              company_name,
              action_timestamp,
              user_session
          ) VALUES (
              'INSERT',
              NEW.customer_id,
              NEW.product_id,
              NEW.company_id,
              NEW.poll_id,
              NULL,
              NEW.rating,
              NEW.daterating,
              v_customer_name,
              v_customer_email,
              v_product_name,
              v_company_name,
              CURRENT_TIMESTAMP,
              USER()
          );
      END//
      
      DELIMITER ;
      ```
      
      ### **3. Impedir insertar productos sin unidad de medida**
      
      ```sql
      //ALTER TABLE products 
      ADD COLUMN unitofmeasure_id INT NULL,
      ADD CONSTRAINT fk_products_unitofmeasure 
          FOREIGN KEY (unitofmeasure_id) REFERENCES unitofmeasure(id) 
          ON DELETE RESTRICT ON UPDATE CASCADE;//
          
      DELIMITER //
      
      CREATE TRIGGER tr_products_validate_unitofmeasure_insert
          BEFORE INSERT ON products
          FOR EACH ROW
      BEGIN
          IF NEW.unitofmeasure_id IS NULL THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = 'Error: No se puede insertar un producto sin unidad de medida. El campo unitofmeasure_id es obligatorio.';
          END IF;
          
          IF NOT EXISTS (SELECT 1 FROM unitofmeasure WHERE id = NEW.unitofmeasure_id) THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = 'Error: La unidad de medida especificada no existe en la tabla unitofmeasure.';
          END IF;
          
          IF NEW.unitofmeasure_id <= 0 THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = 'Error: El ID de unidad de medida debe ser un valor positivo mayor que cero.';
          END IF;
      END//
      
      DELIMITER ;
      
      ```
      
      ### **4. Validar calificaciones no mayores a 5**
      
      ```sql
      DELIMITER //
      
      CREATE TRIGGER tr_validate_rating_insert
          BEFORE INSERT ON quality_products
          FOR EACH ROW
      BEGIN
          DECLARE v_min_rating DOUBLE DEFAULT 1.0;
          DECLARE v_max_rating DOUBLE DEFAULT 5.0;
          DECLARE v_decimal_places INT DEFAULT 1;
          DECLARE v_error_message TEXT;
          DECLARE v_rounded_rating DOUBLE;
          
          SELECT min_rating, max_rating, decimal_places
          INTO v_min_rating, v_max_rating, v_decimal_places
          FROM rating_config 
          WHERE is_active = TRUE 
          ORDER BY id DESC 
          LIMIT 1;
          
          IF NEW.rating IS NULL THEN
              SET v_error_message = 'Error: La calificación no puede ser NULL.';
              
              INSERT INTO rating_validation_log (
                  action_type, customer_id, product_id, company_id, poll_id,
                  attempted_rating, min_allowed, max_allowed, error_message
              ) VALUES (
                  'INSERT', NEW.customer_id, NEW.product_id, NEW.company_id, NEW.poll_id,
                  COALESCE(NEW.rating, 0), v_min_rating, v_max_rating, v_error_message
              );
              
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = v_error_message;
          END IF;
          
          IF NEW.rating < v_min_rating THEN
              SET v_error_message = CONCAT('Error: La calificación (', NEW.rating, ') no puede ser menor que ', v_min_rating, '.');
              
              INSERT INTO rating_validation_log (
                  action_type, customer_id, product_id, company_id, poll_id,
                  attempted_rating, min_allowed, max_allowed, error_message
              ) VALUES (
                  'INSERT', NEW.customer_id, NEW.product_id, NEW.company_id, NEW.poll_id,
                  NEW.rating, v_min_rating, v_max_rating, v_error_message
              );
              
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = v_error_message;
          END IF;
          
          IF NEW.rating > v_max_rating THEN
              SET v_error_message = CONCAT('Error: La calificación (', NEW.rating, ') no puede ser mayor que ', v_max_rating, '.');
              
              INSERT INTO rating_validation_log (
                  action_type, customer_id, product_id, company_id, poll_id,
                  attempted_rating, min_allowed, max_allowed, error_message
              ) VALUES (
                  'INSERT', NEW.customer_id, NEW.product_id, NEW.company_id, NEW.poll_id,
                  NEW.rating, v_min_rating, v_max_rating, v_error_message
              );
              
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = v_error_message;
          END IF;
          
          SET v_rounded_rating = ROUND(NEW.rating, v_decimal_places);
          IF NEW.rating != v_rounded_rating THEN
              SET v_error_message = CONCAT('Error: La calificación debe tener máximo ', v_decimal_places, ' decimal(es). Valor recibido: ', NEW.rating, ', valor permitido: ', v_rounded_rating);
              
              INSERT INTO rating_validation_log (
                  action_type, customer_id, product_id, company_id, poll_id,
                  attempted_rating, min_allowed, max_allowed, error_message
              ) VALUES (
                  'INSERT', NEW.customer_id, NEW.product_id, NEW.company_id, NEW.poll_id,
                  NEW.rating, v_min_rating, v_max_rating, v_error_message
              );
              
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = v_error_message;
          END IF;
          
          IF NEW.rating = 'inf' OR NEW.rating = '-inf' OR NEW.rating != NEW.rating THEN
              SET v_error_message = 'Error: La calificación debe ser un número válido.';
              
              INSERT INTO rating_validation_log (
                  action_type, customer_id, product_id, company_id, poll_id,
                  attempted_rating, min_allowed, max_allowed, error_message
              ) VALUES (
                  'INSERT', NEW.customer_id, NEW.product_id, NEW.company_id, NEW.poll_id,
                  NEW.rating, v_min_rating, v_max_rating, v_error_message
              );
              
              SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = v_error_message;
          END IF;
      END//
      
      DELIMITER ;
      ```
      
      ### **5. Actualizar estado de membresía cuando vence**
      
      ```sql
      DELIMITER //
      
      CREATE TRIGGER tr_membership_check_expiration
          BEFORE UPDATE ON memberships
          FOR EACH ROW
      BEGIN
          DECLARE v_current_date DATE DEFAULT CURDATE();
          DECLARE v_change_reason VARCHAR(255);
          
          IF NEW.expiration_date IS NOT NULL AND NEW.expiration_date < v_current_date THEN
              IF OLD.status = 'ACTIVE' THEN
                  SET NEW.status = 'EXPIRED';
                  SET v_change_reason = CONCAT('Membresía expirada automáticamente el ', NEW.expiration_date);
                  
                  INSERT INTO membership_status_log (
                      membership_id, old_status, new_status, 
                      old_expiration_date, new_expiration_date,
                      change_reason, change_type
                  ) VALUES (
                      NEW.id, OLD.status, NEW.status,
                      OLD.expiration_date, NEW.expiration_date,
                      v_change_reason, 'AUTOMATIC'
                  );
                  
                  INSERT INTO membership_notifications (
                      membership_id, notification_type, scheduled_date
                  ) VALUES (
                      NEW.id, 'EXPIRED', v_current_date
                  );
              END IF;
          END IF;
          
          IF NEW.auto_renew = TRUE AND NEW.status = 'EXPIRED' THEN
              DECLARE v_period_id INT;
              DECLARE v_new_expiration_date DATE;
              
              SELECT period_id INTO v_period_id
              FROM membershipperiods 
              WHERE membership_id = NEW.id 
              ORDER BY membership_id DESC 
              LIMIT 1;
              
              SET v_new_expiration_date = CalculateExpirationDate(NEW.id, v_period_id);
              
              SET NEW.status = 'ACTIVE';
              SET NEW.expiration_date = v_new_expiration_date;
              SET v_change_reason = CONCAT('Renovación automática hasta ', v_new_expiration_date);
              
              INSERT INTO membership_status_log (
                  membership_id, old_status, new_status,
                  old_expiration_date, new_expiration_date,
                  change_reason, change_type
              ) VALUES (
                  NEW.id, 'EXPIRED', 'ACTIVE',
                  OLD.expiration_date, v_new_expiration_date,
                  v_change_reason, 'AUTOMATIC'
              );
              
              INSERT INTO membership_notifications (
                  membership_id, notification_type, scheduled_date
              ) VALUES (
                  NEW.id, 'RENEWED', v_current_date
              );
              
              INSERT INTO membership_notifications (
                  membership_id, notification_type, days_before_expiry, scheduled_date
              ) VALUES 
              (NEW.id, 'EXPIRY_WARNING', 7, DATE_SUB(v_new_expiration_date, INTERVAL 7 DAY)),
              (NEW.id, 'EXPIRY_WARNING', 3, DATE_SUB(v_new_expiration_date, INTERVAL 3 DAY)),
              (NEW.id, 'EXPIRED', 0, v_new_expiration_date);
          END IF;
      END//
      
      DELIMITER ;
      ```
      
      ### **6. Evitar duplicados de productos por empresa**
      
      ```sql
      CREATE TRIGGER trigger_prevent_duplicate_products_insert
          BEFORE INSERT ON companyproducts
          FOR EACH ROW
          EXECUTE FUNCTION check_duplicate_product_in_company();
      ```
      
      ### **7. Enviar notificación al añadir un favorito**
      
      ```sql
      CREATE TRIGGER trigger_favorite_added_notification
          AFTER INSERT ON favorites
          FOR EACH ROW
          EXECUTE FUNCTION send_favorite_notification();
      ```
      
      ### **8. Insertar fila en `quality_products` tras calificación**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_insert_quality_products
      AFTER INSERT ON rates
      FOR EACH ROW
      BEGIN
          INSERT INTO quality_products (
              product_id,
              customer_id,
              poll_id,
              company_id,
              daterating,
              rating
          )
          VALUES (
              (SELECT p.id FROM products p 
               JOIN polls po ON po.categorypoll_id = p.category_id 
               WHERE po.id = NEW.poll_id LIMIT 1),
              NEW.customer_id,
              NEW.poll_id,
              NEW.company_id,
              NEW.daterating,
              NEW.rating
          );
      END$$
      
      DELIMITER ;
      ```
      
      ### **9. Eliminar favoritos si se elimina el producto**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_delete_product_favorites
      BEFORE DELETE ON products
      FOR EACH ROW
      BEGIN
          DELETE FROM favorites 
          WHERE company_id = OLD.id;
          
          DELETE FROM details_favorites 
          WHERE product_id = OLD.id;
      END$$
      
      DELIMITER ;
      ```
      
      ### **10. Bloquear modificación de audiencias activas**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_block_active_audience_update
      BEFORE UPDATE ON audiences
      FOR EACH ROW
      BEGIN
          DECLARE v_active_customers INT DEFAULT 0;
          DECLARE v_active_companies INT DEFAULT 0;
          DECLARE v_active_benefits INT DEFAULT 0;
          
          SELECT COUNT(*) INTO v_active_customers
          FROM customers
          WHERE audience_id = OLD.id;
          
          SELECT COUNT(*) INTO v_active_companies
          FROM companies
          WHERE audience_id = OLD.id;
          
          SELECT COUNT(*) INTO v_active_benefits
          FROM membershipbenefits mb
          JOIN memberships m ON mb.membership_id = m.id
          WHERE mb.audience_id = OLD.id;
          
          IF (v_active_customers > 0 OR v_active_companies > 0 OR v_active_benefits > 0) THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = 'ERROR: No se puede modificar una audiencia activa. La audiencia tiene registros relacionados activos.';
          END IF;
      END$$
      
      DELIMITER ;
      ```
      
      ### **11. Recalcular promedio de calidad del producto tras nueva evaluación**
      
      ```sql
      //ALTER TABLE products ADD COLUMN IF NOT EXISTS average_rating DECIMAL(3,2) DEFAULT 0.00;
      ALTER TABLE products ADD COLUMN IF NOT EXISTS total_ratings INT DEFAULT 0;//
      
      DELIMITER $$
      
      CREATE TRIGGER tr_update_product_quality_average
      AFTER INSERT ON quality_products
      FOR EACH ROW
      BEGIN
          DECLARE v_avg_rating DECIMAL(5,2);
          DECLARE v_total_ratings INT;
          
          SELECT 
              AVG(rating) as avg_rating,
              COUNT(*) as total_ratings
          INTO v_avg_rating, v_total_ratings
          FROM quality_products
          WHERE product_id = NEW.product_id;
          
          UPDATE products 
          SET 
              average_rating = v_avg_rating,
              total_ratings = v_total_ratings
          WHERE id = NEW.product_id;
          
      END$$
      
      DELIMITER ;
      ```
      
      ### **12. Registrar asignación de nuevo beneficio**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_audit_benefit_assignment_insert
      AFTER INSERT ON membershipbenefits
      FOR EACH ROW
      BEGIN
          DECLARE v_benefit_description VARCHAR(800);
          DECLARE v_membership_name VARCHAR(50);
          DECLARE v_period_name VARCHAR(50);
          
          SELECT description INTO v_benefit_description 
          FROM benefits 
          WHERE id = NEW.benefit_id;
          
          SELECT name INTO v_membership_name 
          FROM memberships 
          WHERE id = NEW.membership_id;
          
          SELECT name INTO v_period_name 
          FROM periods 
          WHERE id = NEW.period_id;
          
          INSERT INTO audit_benefit_assignments (
              membership_id,
              period_id,
              audience_id,
              benefit_id,
              action_type,
              new_values,
              user_session,
              notes
          ) VALUES (
              NEW.membership_id,
              NEW.period_id,
              NEW.audience_id,
              NEW.benefit_id,
              'INSERT',
              JSON_OBJECT(
                  'membership_id', NEW.membership_id,
                  'period_id', NEW.period_id,
                  'audience_id', NEW.audience_id,
                  'benefit_id', NEW.benefit_id,
                  'membership_name', v_membership_name,
                  'period_name', v_period_name,
                  'benefit_description', v_benefit_description
              ),
              COALESCE(CONNECTION_ID(), 'SYSTEM'),
              CONCAT('Nuevo beneficio asignado: ', v_benefit_description, 
                     ' para membresía: ', v_membership_name,
                     ' en período: ', v_period_name)
          );
      END$$
      
      DELIMITER ;
      ```
      
      ### **13. Impedir doble calificación por parte del cliente**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_prevent_duplicate_product_rating
      BEFORE INSERT ON quality_products
      FOR EACH ROW
      BEGIN
          DECLARE v_existing_rating_count INT DEFAULT 0;
          DECLARE v_last_rating_date DATETIME;
          DECLARE v_time_diff_hours INT;
          
          SELECT COUNT(*), MAX(daterating)
          INTO v_existing_rating_count, v_last_rating_date
          FROM quality_products
          WHERE customer_id = NEW.customer_id 
          AND product_id = NEW.product_id;
          
          IF v_existing_rating_count > 0 THEN
              SET v_time_diff_hours = TIMESTAMPDIFF(HOUR, v_last_rating_date, NEW.daterating);
              
              IF v_time_diff_hours < 24 THEN
                  SIGNAL SQLSTATE '45000' 
                  SET MESSAGE_TEXT = 'ERROR: No puede calificar el mismo producto dos veces en menos de 24 horas. Espere al menos 24 horas desde su última calificación.';
              END IF;
          END IF;
      END$$
      
      DELIMITER ;
      
      ```
      
      ### **14. Validar correos duplicados en clientes**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_validate_unique_email_insert
      BEFORE INSERT ON customers
      FOR EACH ROW
      BEGIN
          DECLARE v_email_count INT DEFAULT 0;
          DECLARE v_existing_customer_name VARCHAR(60);
          
          SET NEW.email = LOWER(TRIM(NEW.email));
          
          SELECT COUNT(*), name
          INTO v_email_count, v_existing_customer_name
          FROM customers
          WHERE LOWER(TRIM(email)) = NEW.email
          LIMIT 1;
          
          IF v_email_count > 0 THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = CONCAT('ERROR: El email "', NEW.email, '" ya está registrado por otro cliente: ', v_existing_customer_name);
          END IF;
          
          IF NEW.email NOT REGEXP '^[A-Za-z0-9._%-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = CONCAT('ERROR: El formato del email "', NEW.email, '" no es válido.');
          END IF;
      END$$
      
      DELIMITER ;
      ```
      
      ### **15. Eliminar detalles de favoritos huérfanos**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_cleanup_orphan_details_on_favorite_delete
      AFTER DELETE ON favorites
      FOR EACH ROW
      BEGIN
          DELETE FROM details_favorites 
          WHERE favorite_id = OLD.id;
      END$$
      
      DELIMITER ;
      ```
      
      ### **16. Actualizar campo `updated_at` en `companies`**
      
      ```sql
      //ALTER TABLE companies 
      ADD COLUMN IF NOT EXISTS created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
      ADD COLUMN IF NOT EXISTS updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP;//
      
      DELIMITER $$
      
      CREATE TRIGGER tr_update_companies_timestamp
      BEFORE UPDATE ON companies
      FOR EACH ROW
      BEGIN
          SET NEW.updated_at = CURRENT_TIMESTAMP;
      END$$
      
      DELIMITER ;
      ```
      
      ### **17. Impedir borrar ciudad si hay empresas activas**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_prevent_city_deletion_with_active_companies
      BEFORE DELETE ON citiesormunicipalities
      FOR EACH ROW
      BEGIN
          DECLARE v_active_companies INT DEFAULT 0;
          DECLARE v_active_customers INT DEFAULT 0;
          DECLARE v_company_names TEXT DEFAULT '';
          
          SELECT COUNT(*) INTO v_active_companies
          FROM companies
          WHERE city_id = OLD.code;
          
          SELECT COUNT(*) INTO v_active_customers
          FROM customers
          WHERE city_id = OLD.code;
          
          IF v_active_companies > 0 THEN
              SELECT GROUP_CONCAT(name SEPARATOR ', ') INTO v_company_names
              FROM (
                  SELECT name FROM companies 
                  WHERE city_id = OLD.code 
                  LIMIT 5
              ) AS company_sample;
              
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = CONCAT(
                  'ERROR: No se puede eliminar la ciudad "', OLD.name, 
                  '" porque tiene ', v_active_companies, ' empresa(s) activa(s). ',
                  'Empresas: ', v_company_names,
                  IF(v_active_companies > 5, ' y otras más...', '')
              );
          END IF;
          
          IF v_active_customers > 0 THEN
              SIGNAL SQLSTATE '45000' 
              SET MESSAGE_TEXT = CONCAT(
                  'ERROR: No se puede eliminar la ciudad "', OLD.name, 
                  '" porque tiene ', v_active_customers, ' cliente(s) activo(s). ',
                  'Primero debe reasignar o eliminar los clientes de esta ciudad.'
              );
          END IF;
      END$$
      
      DELIMITER ;
      ```
      
      ### **18. Registrar cambios de estado en encuestas**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_audit_poll_status_insert
      AFTER INSERT ON polls
      FOR EACH ROW
      BEGIN
          INSERT INTO audit_poll_status_changes (
              poll_id,
              poll_name,
              old_status,
              new_status,
              old_isactive,
              new_isactive,
              changed_by,
              session_id,
              change_reason,
              additional_info
          ) VALUES (
              NEW.id,
              NEW.name,
              NULL,
              COALESCE(NEW.status, 'draft'),
              NULL,
              NEW.isactive,
              USER(),
              CONNECTION_ID(),
              'Nueva encuesta creada',
              JSON_OBJECT(
                  'categorypoll_id', NEW.categorypoll_id,
                  'description', LEFT(NEW.description, 100),
                  'created_at', NOW()
              )
          );
      END$$
      
      DELIMITER ;
      ```
      
      ### **19. Sincronizar `rates` y `quality_products`**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_sync_rates_to_quality_insert
      AFTER INSERT ON rates
      FOR EACH ROW
      BEGIN
          DECLARE v_product_id INT;
          DECLARE v_existing_quality_record INT DEFAULT 0;
          
          SELECT cp.product_id INTO v_product_id
          FROM companyproducts cp
          JOIN polls p ON p.categorypoll_id = (
              SELECT category_id FROM products 
              WHERE id = cp.product_id LIMIT 1
          )
          WHERE cp.company_id = NEW.company_id
          AND p.id = NEW.poll_id
          LIMIT 1;
          
          IF v_product_id IS NOT NULL THEN
              SELECT COUNT(*) INTO v_existing_quality_record
              FROM quality_products
              WHERE customer_id = NEW.customer_id
              AND company_id = NEW.company_id
              AND poll_id = NEW.poll_id
              AND product_id = v_product_id;
              
              IF v_existing_quality_record = 0 THEN
                  INSERT INTO quality_products (
                      product_id,
                      customer_id,
                      poll_id,
                      company_id,
                      daterating,
                      rating
                  ) VALUES (
                      v_product_id,
                      NEW.customer_id,
                      NEW.poll_id,
                      NEW.company_id,
                      NEW.daterating,
                      NEW.rating
                  );
              ELSE
                  UPDATE quality_products
                  SET 
                      daterating = NEW.daterating,
                      rating = NEW.rating
                  WHERE customer_id = NEW.customer_id
                  AND company_id = NEW.company_id
                  AND poll_id = NEW.poll_id
                  AND product_id = v_product_id;
              END IF;
          END IF;
      END$$
      
      DELIMITER ;
      ```
      
      ### **20. Eliminar productos sin relación a empresas**
      
      ```sql
      DELIMITER $$
      
      CREATE TRIGGER tr_delete_orphan_products
      AFTER DELETE ON companyproducts
      FOR EACH ROW
      BEGIN
          DELETE FROM products 
          WHERE id = OLD.product_id 
          AND id NOT IN (
              SELECT DISTINCT product_id 
              FROM companyproducts 
              WHERE product_id = OLD.product_id
          );
      END$$
      
      DELIMITER ;
      ```
      
      
      
      ## **6. Events (Eventos Programados..Usar procedimientos o funciones para cada evento)**
      
      ## 🔹 **1. Borrar productos sin actividad cada 6 meses**
      
      ```sql
      DELIMITER $$
      
      CREATE EVENT IF NOT EXISTS ev_delete_inactive_products
      ON SCHEDULE EVERY 6 MONTH
      STARTS CURRENT_TIMESTAMP + INTERVAL 6 MONTH
      DO
      BEGIN
          DECLARE products_deleted INT DEFAULT 0;
          DECLARE log_message VARCHAR(255);
          
          CREATE TEMPORARY TABLE temp_inactive_products AS
          SELECT DISTINCT p.id, p.name
          FROM products p
          WHERE p.id NOT IN (
              SELECT DISTINCT df.product_id 
              FROM details_favorites df
              INNER JOIN favorites f ON df.favorite_id = f.id
              WHERE f.id IN (
                  SELECT id FROM favorites 
                  WHERE id > 0
              )
              
              UNION
              
              SELECT DISTINCT qp.product_id
              FROM quality_products qp
              WHERE qp.daterating >= DATE_SUB(NOW(), INTERVAL 6 MONTH)
              
              UNION
              
              SELECT DISTINCT cp.product_id
              FROM companyproducts cp
              INNER JOIN companies c ON cp.company_id = c.id
              WHERE c.id IS NOT NULL
          );
          
          SELECT COUNT(*) INTO products_deleted FROM temp_inactive_products;
          
          DELETE df FROM details_favorites df
          INNER JOIN temp_inactive_products tip ON df.product_id = tip.id;
          
          DELETE qp FROM quality_products qp
          INNER JOIN temp_inactive_products tip ON qp.product_id = tip.id;
          
          DELETE cp FROM companyproducts cp
          INNER JOIN temp_inactive_products tip ON cp.product_id = tip.id;
          
          DELETE p FROM products p
          INNER JOIN temp_inactive_products tip ON p.id = tip.id;
          
          SET log_message = CONCAT('Evento ejecutado: ', products_deleted, ' productos inactivos eliminados en ', NOW());
          
          CREATE TABLE IF NOT EXISTS event_logs (
              id INT AUTO_INCREMENT PRIMARY KEY,
              event_name VARCHAR(100),
              execution_time DATETIME,
              message TEXT,
              records_affected INT
          );
          
          INSERT INTO event_logs (event_name, execution_time, message, records_affected)
          VALUES ('ev_delete_inactive_products', NOW(), log_message, products_deleted);
          
          DROP TEMPORARY TABLE temp_inactive_products;
          
      END$$
      
      DELIMITER ;
      ```
      
      ## 🔹 **2. Recalcular el promedio de calificaciones semanalmente**
      
      ```sql
      DELIMITER $$
      
      CREATE EVENT IF NOT EXISTS ev_weekly_rating_recalculation
      ON SCHEDULE EVERY 1 WEEK
      STARTS NEXT_SUNDAY + INTERVAL 2 HOUR
      DO
      BEGIN
          DECLARE start_time DATETIME;
          DECLARE end_time DATETIME;
          DECLARE products_updated INT DEFAULT 0;
          DECLARE total_ratings INT DEFAULT 0;
          DECLARE duration_seconds INT;
          DECLARE error_occurred BOOLEAN DEFAULT FALSE;
          DECLARE error_msg TEXT DEFAULT '';
          
          DECLARE CONTINUE HANDLER FOR SQLEXCEPTION 
          BEGIN
              SET error_occurred = TRUE;
              GET DIAGNOSTICS CONDITION 1
                  error_msg = MESSAGE_TEXT;
          END;
          
          SET start_time = NOW();
          
          DELETE FROM product_rating_summary;
          
          INSERT INTO product_rating_summary (
              product_id, 
              average_rating, 
              total_ratings, 
              last_updated,
              rating_distribution
          )
          SELECT 
              p.id as product_id,
              COALESCE(ROUND(AVG(qp.rating), 2), 0) as average_rating,
              COUNT(qp.rating) as total_ratings,
              NOW() as last_updated,
              JSON_OBJECT(
                  'rating_1', SUM(CASE WHEN qp.rating = 1 THEN 1 ELSE 0 END),
                  'rating_2', SUM(CASE WHEN qp.rating = 2 THEN 1 ELSE 0 END),
                  'rating_3', SUM(CASE WHEN qp.rating = 3 THEN 1 ELSE 0 END),
                  'rating_4', SUM(CASE WHEN qp.rating = 4 THEN 1 ELSE 0 END),
                  'rating_5', SUM(CASE WHEN qp.rating = 5 THEN 1 ELSE 0 END)
              ) as rating_distribution
          FROM products p
          LEFT JOIN quality_products qp ON p.id = qp.product_id
          GROUP BY p.id;
          
          SELECT COUNT(*) INTO products_updated FROM product_rating_summary;
          SELECT SUM(total_ratings) INTO total_ratings FROM product_rating_summary;
          
          SET end_time = NOW();
          SET duration_seconds = TIMESTAMPDIFF(SECOND, start_time, end_time);
          
          INSERT INTO rating_recalculation_logs (
              execution_time,
              products_updated,
              total_ratings_processed,
              execution_duration_seconds,
              status,
              error_message
          ) VALUES (
              start_time,
              products_updated,
              total_ratings,
              duration_seconds,
              CASE WHEN error_occurred THEN 'ERROR' ELSE 'SUCCESS' END,
              error_msg
          );
          
      END$$
      
      DELIMITER ;
      ```
      
      ## 🔹 **3. Actualizar precios según inflación mensual**
      
      ```sql
      CREATE EVENT IF NOT EXISTS actualizar_precios_inflacion
      ON SCHEDULE EVERY 1 MONTH
      DO
        UPDATE company_products
        SET price = price * 1.03;
      ```
      
      ## 🔹 **4. Crear backups lógicos diariamente**
      
      ```sql
      CREATE EVENT IF NOT EXISTS backup_diario
      ON SCHEDULE EVERY 1 DAY STARTS CURRENT_DATE + INTERVAL 1 DAY
      DO
        INSERT INTO products_backup (SELECT * FROM products);
      ```
      
      ## 🔹 **5. Notificar sobre productos favoritos sin calificar**
      
      ```sql
      CREATE EVENT IF NOT EXISTS notificar_favoritos_sin_calificar
      ON SCHEDULE EVERY 1 WEEK
      DO
        INSERT INTO user_reminders (customer_id, product_id)
        SELECT f.customer_id, df.product_id
        FROM favorites f
        JOIN details_favorites df ON f.id = df.favorite_id
        LEFT JOIN rates r ON r.product_id = df.product_id AND r.customer_id = f.customer_id
        WHERE r.id IS NULL;
      
      ```
      
      ## 🔹 **6. Revisar inconsistencias entre empresa y productos**
      
      ```sql
      CREATE EVENT IF NOT EXISTS revisar_inconsistencias
      ON SCHEDULE EVERY 1 WEEK
      DO
        INSERT INTO errores_log(descripcion)
        SELECT CONCAT('Producto sin empresa: ', p.name)
        FROM products p
        WHERE NOT EXISTS (SELECT 1 FROM company_products cp WHERE cp.product_id = p.id);
      ```
      
      ## 🔹 **7. Archivar membresías vencidas diariamente**
      
      ```sql
      CREATE EVENT IF NOT EXISTS archivar_membresias
      ON SCHEDULE EVERY 1 DAY
      DO
        UPDATE membership_periods
        SET status = 'INACTIVA'
        WHERE end_date < CURDATE();
      ```
      
      ## 🔹 **8. Notificar beneficios nuevos a usuarios semanalmente**
      
      ```sql
      CREATE EVENT IF NOT EXISTS notificar_nuevos_beneficios
      ON SCHEDULE EVERY 1 WEEK
      DO
        INSERT INTO notificaciones (mensaje, created_at)
        SELECT CONCAT('Nuevo beneficio disponible: ', name), NOW()
        FROM benefits
        WHERE created_at >= NOW() - INTERVAL 7 DAY;
      
      ```
      
      ## 🔹 **9. Calcular cantidad de favoritos por cliente mensualmente**
      
      ```sql
      CREATE EVENT IF NOT EXISTS resumen_favoritos_clientes
      ON SCHEDULE EVERY 1 MONTH
      DO
        INSERT INTO favoritos_resumen (customer_id, total)
        SELECT f.customer_id, COUNT(df.product_id)
        FROM favorites f
        JOIN details_favorites df ON f.id = df.favorite_id
        GROUP BY f.customer_id;
      ```
      
      ## 🔹 **10. Validar claves foráneas semanalmente**
      
      ```sql
      CREATE EVENT IF NOT EXISTS validar_fk_rates
      ON SCHEDULE EVERY 1 WEEK
      DO
        INSERT INTO inconsistencias_fk (descripcion)
        SELECT CONCAT('Rate con poll inexistente ID: ', r.id)
        FROM rates r
        LEFT JOIN polls p ON r.poll_id = p.id
        WHERE p.id IS NULL;
      
      ```
      
      ## 🔹 **11. Eliminar calificaciones inválidas antiguas**
      
      ```sql
      CREATE EVENT IF NOT EXISTS eliminar_rates_invalidas
      ON SCHEDULE EVERY 1 MONTH
      DO
        DELETE FROM rates
        WHERE (rating IS NULL OR rating < 0)
          AND created_at < NOW() - INTERVAL 3 MONTH;
      ```
      
      ## 🔹 **12. Cambiar estado de encuestas inactivas automáticamente**
      
      ```sql
      CREATE EVENT IF NOT EXISTS inactivar_encuestas
      ON SCHEDULE EVERY 1 MONTH
      DO
        UPDATE polls
        SET status = 'INACTIVA'
        WHERE id NOT IN (
          SELECT DISTINCT poll_id FROM rates
          WHERE created_at >= NOW() - INTERVAL 6 MONTH
        );
      ```
      
      ## 🔹 **13. Registrar auditorías de forma periódica**
      
      ```sql
      CREATE EVENT IF NOT EXISTS registrar_auditorias
      ON SCHEDULE EVERY 1 DAY
      DO
        INSERT INTO auditorias_diarias (fecha, total_productos, total_usuarios)
        VALUES (CURDATE(), (SELECT COUNT(*) FROM products), (SELECT COUNT(*) FROM customers));
      ```
      
      ## 🔹 **14. Notificar métricas de calidad a empresas**
      
      ```sql
      CREATE EVENT IF NOT EXISTS notificar_metricas_calidad
      ON SCHEDULE EVERY 1 WEEK STARTS CURRENT_DATE + INTERVAL 1 WEEK
      DO
        INSERT INTO notificaciones_empresa (empresa_id, mensaje)
        SELECT c.id, CONCAT('Su producto ', p.name, ' tiene promedio ', ROUND(AVG(q.rating),2))
        FROM companies c
        JOIN company_products cp ON c.id = cp.company_id
        JOIN products p ON cp.product_id = p.id
        JOIN quality_products q ON q.product_id = p.id
        GROUP BY c.id, p.name;
      ```
      
      ## 🔹 **15. Recordar renovación de membresías**
      
      ```sql
      CREATE EVENT IF NOT EXISTS recordar_renovacion_membresias
      ON SCHEDULE EVERY 1 WEEK
      DO
        INSERT INTO recordatorios (customer_id, mensaje)
        SELECT customer_id, 'Tu membresía está próxima a vencer'
        FROM membership_periods
        WHERE end_date BETWEEN CURDATE() AND CURDATE() + INTERVAL 7 DAY;
      ```
      
      ## 🔹 **16. Reordenar estadísticas generales cada semana**
      
      ```sql
      CREATE EVENT IF NOT EXISTS actualizar_estadisticas_generales
      ON SCHEDULE EVERY 1 WEEK
      DO
        UPDATE estadisticas
        SET total_productos = (SELECT COUNT(*) FROM products),
            total_empresas = (SELECT COUNT(*) FROM companies),
            total_clientes = (SELECT COUNT(*) FROM customers);
      ```
      
      ## 🔹 **17. Crear resúmenes temporales de uso por categoría**
      
      ```sql
      CREATE EVENT IF NOT EXISTS resumen_uso_por_categoria
      ON SCHEDULE EVERY 1 MONTH
      DO
        INSERT INTO resumen_categoria (categoria_id, total)
        SELECT category_id, COUNT(*)
        FROM products
        JOIN quality_products USING(product_id)
        GROUP BY category_id;
      ```
      
      ## 🔹 **18. Actualizar beneficios caducados**
      
      ```sql
      CREATE EVENT IF NOT EXISTS desactivar_beneficios_expirados
      ON SCHEDULE EVERY 1 DAY
      DO
        UPDATE benefits
        SET status = 'INACTIVO'
        WHERE expires_at IS NOT NULL AND expires_at < CURDATE();
      ```
      
      ## 🔹 **19. Alertar productos sin evaluación anual**
      
      ```sql
      CREATE EVENT IF NOT EXISTS alertar_sin_evaluacion
      ON SCHEDULE EVERY 1 MONTH
      DO
        INSERT INTO alertas_productos(product_id, mensaje)
        SELECT id, 'Producto sin evaluación anual'
        FROM products
        WHERE id NOT IN (
          SELECT DISTINCT product_id FROM quality_products
          WHERE date_rating >= NOW() - INTERVAL 1 YEAR
        );
      ```
      
      ## 🔹 **20. Actualizar precios con índice externo**
      
      ```sql
      CREATE EVENT IF NOT EXISTS actualizar_precio_por_indice
      ON SCHEDULE EVERY 1 MONTH
      DO
        UPDATE company_products cp
        JOIN inflacion_indice ii ON 1=1
        SET cp.price = cp.price * ii.valor
        WHERE ii.fecha = (SELECT MAX(fecha) FROM inflacion_indice);
      ```
      
      
      
      ## 🔹 **7. Historias de Usuario con JOINs**
      
      ## 🔹 **1. Ver productos con la empresa que los vende**
      
      ```sql
      SELECT 
          c.id AS empresa_id,
          c.name AS empresa_nombre,
          c.email AS empresa_email,
          c.cellphone AS empresa_telefono,
          p.id AS producto_id,
          p.name AS producto_nombre,
          p.detail AS producto_detalle,
          cp.price AS precio_empresa,
          p.price AS precio_base_producto,
          cat.description AS categoria_producto,
          um.description AS unidad_medida
      FROM companies c
      LEFT JOIN companyproducts cp ON c.id = cp.company_id
      LEFT JOIN products p ON cp.product_id = p.id
      LEFT JOIN categories cat ON p.category_id = cat.id
      LEFT JOIN unitofmeasure um ON cp.unitmeasure_id = um.id
      ORDER BY c.name, p.name;
      ```
      
      ## 🔹 **2. Mostrar productos favoritos con su empresa y categoría**
      
      ```sql
      SELECT 
          cust.id AS cliente_id,
          cust.name AS cliente_nombre,
          comp.name AS empresa_nombre,
          comp.email AS empresa_email,
          comp.cellphone AS empresa_telefono,
          p.name AS producto_nombre,
          p.detail AS producto_detalle,
          cat.description AS categoria_producto,
          cp.price AS precio_empresa,
          p.price AS precio_base,
          um.description AS unidad_medida,
          p.image AS imagen_producto
      FROM customers cust
      INNER JOIN favorites f ON cust.id = f.customer_id
      INNER JOIN companies comp ON f.company_id = comp.id
      INNER JOIN companyproducts cp ON comp.id = cp.company_id
      INNER JOIN products p ON cp.product_id = p.id
      INNER JOIN categories cat ON p.category_id = cat.id
      LEFT JOIN unitofmeasure um ON cp.unitmeasure_id = um.id
      WHERE cust.id = 1
      ORDER BY comp.name, cat.description, p.name;
      ```
      
      ## 🔹 **3. Ver empresas aunque no tengan productos**
      
      ```sql
      SELECT 
          c.id AS empresa_id,
          c.name AS empresa_nombre,
          c.email AS empresa_email,
          c.cellphone AS empresa_telefono,
          city.name AS ciudad,
          state.name AS estado,
          country.name AS pais,
          cat_emp.description AS categoria_empresa,
          aud.description AS audiencia,
          COALESCE(p.name, 'Sin productos') AS producto_nombre,
          COALESCE(p.detail, 'N/A') AS producto_detalle,
          COALESCE(cp.price, 0) AS precio_empresa,
          COALESCE(p.price, 0) AS precio_base,
          COALESCE(cat_prod.description, 'N/A') AS categoria_producto,
          COALESCE(um.description, 'N/A') AS unidad_medida,
          CASE 
              WHEN cp.product_id IS NULL THEN 'Sin productos'
              ELSE 'Con productos'
          END AS estado_productos
      FROM companies c
      LEFT JOIN companyproducts cp ON c.id = cp.company_id
      LEFT JOIN products p ON cp.product_id = p.id
      LEFT JOIN categories cat_prod ON p.category_id = cat_prod.id
      LEFT JOIN categories cat_emp ON c.category_id = cat_emp.id
      LEFT JOIN unitofmeasure um ON cp.unitmeasure_id = um.id
      LEFT JOIN citiesormunicipalities city ON c.city_id = city.code
      LEFT JOIN stateorregions state ON city.statereg_id = state.code
      LEFT JOIN countries country ON state.country_id = country.isocode
      LEFT JOIN audiences aud ON c.audience_id = aud.id
      ORDER BY c.name, p.name;
      ```
      
      ## 🔹 **4. Ver productos que fueron calificados (o no)**
      
      ```sql
      SELECT 
          p.id AS producto_id,
          p.name AS producto_nombre,
          p.detail AS producto_detalle,
          p.price AS precio_base,
          p.image AS imagen_producto,
          cat.description AS categoria_producto,
          comp.name AS empresa_nombre,
          comp.email AS empresa_email,
          cust.name AS cliente_nombre,
          cust.email AS cliente_email,
          polls.name AS encuesta_nombre,
          polls.description AS encuesta_descripcion,
          COALESCE(qp.rating, 0) AS calificacion,
          COALESCE(qp.daterating, NULL) AS fecha_calificacion,
          CASE 
              WHEN qp.rating IS NULL THEN 'Sin calificar'
              WHEN qp.rating >= 4.5 THEN 'Excelente'
              WHEN qp.rating >= 3.5 THEN 'Bueno'
              WHEN qp.rating >= 2.5 THEN 'Regular'
              WHEN qp.rating >= 1.5 THEN 'Malo'
              ELSE 'Muy malo'
          END AS clasificacion_rating,
          CASE 
              WHEN qp.product_id IS NULL THEN 'No calificado'
              ELSE 'Calificado'
          END AS estado_calificacion
      FROM products p
      LEFT JOIN quality_products qp ON p.id = qp.product_id
      LEFT JOIN customers cust ON qp.customer_id = cust.id
      LEFT JOIN companies comp ON qp.company_id = comp.id
      LEFT JOIN polls ON qp.poll_id = polls.id
      LEFT JOIN categories cat ON p.category_id = cat.id
      ORDER BY p.name, qp.daterating DESC;
      ```
      
      ## 🔹 **5. Ver productos con promedio de calificación y empresa**
      
      ```sql
      SELECT 
          p.id AS producto_id,
          p.name AS producto_nombre,
          p.detail AS producto_detalle,
          p.price AS precio_base,
          cat.description AS categoria_producto,
          comp.name AS empresa_nombre,
          comp.email AS empresa_email,
          comp.cellphone AS empresa_telefono,
          cp.price AS precio_empresa,
          COUNT(qp.rating) AS total_calificaciones,
          COALESCE(ROUND(AVG(qp.rating), 2), 0) AS promedio_calificacion,
          COALESCE(MIN(qp.rating), 0) AS calificacion_minima,
          COALESCE(MAX(qp.rating), 0) AS calificacion_maxima,
          CASE 
              WHEN COUNT(qp.rating) = 0 THEN 'Sin calificar'
              WHEN AVG(qp.rating) >= 4.5 THEN 'Excelente'
              WHEN AVG(qp.rating) >= 3.5 THEN 'Bueno'
              WHEN AVG(qp.rating) >= 2.5 THEN 'Regular'
              WHEN AVG(qp.rating) >= 1.5 THEN 'Malo'
              ELSE 'Muy malo'
          END AS clasificacion_promedio,
          CASE 
              WHEN COUNT(qp.rating) = 0 THEN 'Nuevo producto'
              WHEN COUNT(qp.rating) < 5 THEN 'Pocas calificaciones'
              WHEN COUNT(qp.rating) < 20 THEN 'Calificaciones moderadas'
              ELSE 'Bien calificado'
          END AS estado_calificaciones
      FROM products p
      INNER JOIN companyproducts cp ON p.id = cp.product_id
      INNER JOIN companies comp ON cp.company_id = comp.id
      LEFT JOIN quality_products qp ON p.id = qp.product_id AND comp.id = qp.company_id
      LEFT JOIN categories cat ON p.category_id = cat.id
      GROUP BY 
          p.id, p.name, p.detail, p.price, cat.description, 
          comp.name, comp.email, comp.cellphone, cp.price
      ORDER BY promedio_calificacion DESC, total_calificaciones DESC, p.name;
      ```
      
      ## 🔹 **6. Ver clientes y sus calificaciones (si las tienen)**
      
      ```sql
      SELECT 
          cust.id AS cliente_id,
          cust.name AS cliente_nombre,
          cust.email AS cliente_email,
          cust.cellphone AS cliente_telefono,
          cust.address AS cliente_direccion,
          city.name AS ciudad_cliente,
          state.name AS estado_cliente,
          country.name AS pais_cliente,
          aud.description AS audiencia_cliente,
          p.name AS producto_calificado,
          p.detail AS producto_detalle,
          cat.description AS categoria_producto,
          comp.name AS empresa_calificada,
          comp.email AS empresa_email,
          polls.name AS encuesta_nombre,
          COALESCE(qp.rating, 0) AS calificacion,
          qp.daterating AS fecha_calificacion,
          CASE 
              WHEN qp.rating IS NULL THEN 'Sin calificaciones'
              WHEN qp.rating >= 4.5 THEN 'Excelente'
              WHEN qp.rating >= 3.5 THEN 'Bueno'
              WHEN qp.rating >= 2.5 THEN 'Regular'
              WHEN qp.rating >= 1.5 THEN 'Malo'
              ELSE 'Muy malo'
          END AS clasificacion_rating,
          CASE 
              WHEN qp.product_id IS NULL THEN 'Cliente inactivo'
              ELSE 'Cliente activo'
          END AS estado_cliente
      FROM customers cust
      LEFT JOIN quality_products qp ON cust.id = qp.customer_id
      LEFT JOIN products p ON qp.product_id = p.id
      LEFT JOIN companies comp ON qp.company_id = comp.id
      LEFT JOIN polls ON qp.poll_id = polls.id
      LEFT JOIN categories cat ON p.category_id = cat.id
      LEFT JOIN citiesormunicipalities city ON cust.city_id = city.code
      LEFT JOIN stateorregions state ON city.statereg_id = state.code
      LEFT JOIN countries country ON state.country_id = country.isocode
      LEFT JOIN audiences aud ON cust.audience_id = aud.id
      ORDER BY cust.name, qp.daterating DESC;
      ```
      
      ## 🔹 **7. Ver favoritos con la última calificación del cliente**
      
      ```sql
      SELECT 
          cust.id AS cliente_id,
          cust.name AS cliente_nombre,
          cust.email AS cliente_email,
          comp.id AS empresa_id,
          comp.name AS empresa_nombre,
          comp.email AS empresa_email,
          comp.cellphone AS empresa_telefono,
          city.name AS ciudad_empresa,
          cat_emp.description AS categoria_empresa,
          aud.description AS audiencia_empresa,
          p.name AS ultimo_producto_calificado,
          p.detail AS detalle_producto,
          cat_prod.description AS categoria_producto,
          qp_last.rating AS ultima_calificacion,
          qp_last.daterating AS fecha_ultima_calificacion,
          polls.name AS encuesta_utilizada,
          CASE 
              WHEN qp_last.rating IS NULL THEN 'Sin calificaciones'
              WHEN qp_last.rating >= 4.5 THEN 'Excelente'
              WHEN qp_last.rating >= 3.5 THEN 'Bueno'
              WHEN qp_last.rating >= 2.5 THEN 'Regular'
              WHEN qp_last.rating >= 1.5 THEN 'Malo'
              ELSE 'Muy malo'
          END AS clasificacion_ultima_calificacion,
          CASE 
              WHEN qp_last.rating IS NULL THEN 'No has calificado esta empresa'
              ELSE 'Empresa calificada'
          END AS estado_calificacion
      FROM customers cust
      INNER JOIN favorites f ON cust.id = f.customer_id
      INNER JOIN companies comp ON f.company_id = comp.id
      LEFT JOIN citiesormunicipalities city ON comp.city_id = city.code
      LEFT JOIN categories cat_emp ON comp.category_id = cat_emp.id
      LEFT JOIN audiences aud ON comp.audience_id = aud.id
      LEFT JOIN LATERAL (
          SELECT 
              qp.rating,
              qp.daterating,
              qp.product_id,
              qp.poll_id
          FROM quality_products qp
          WHERE qp.customer_id = cust.id 
          AND qp.company_id = comp.id
          ORDER BY qp.daterating DESC
          LIMIT 1
      ) qp_last ON true
      LEFT JOIN products p ON qp_last.product_id = p.id
      LEFT JOIN categories cat_prod ON p.category_id = cat_prod.id
      LEFT JOIN polls ON qp_last.poll_id = polls.id
      WHERE cust.id = 1
      ORDER BY comp.name;
      ```
      
      ## 🔹 **8. Ver beneficios incluidos en cada plan de membresía**
      
      ```sql
      SELECT 
          m.id as membership_id,
          m.name as membership_name,
          m.description as membership_description,
          b.id as benefit_id,
          b.description as benefit_description,
          b.detail as benefit_detail,
          mb.period_id,
          mb.audience_id
      FROM membershipbenefits mb
      INNER JOIN memberships m ON mb.membership_id = m.id
      INNER JOIN benefits b ON mb.benefit_id = b.id
      ORDER BY m.name, b.description;
      ```
      
      ## 🔹 **9. Ver clientes con membresía activa y sus beneficios**
      
      ```sql
      SELECT 
          c.id as customer_id,
          c.name as customer_name,
          c.email as customer_email,
          c.cellphone as customer_phone,
          c.address as customer_address,
          m.id as membership_id,
          m.name as membership_name,
          m.description as membership_description,
          p.id as period_id,
          p.name as period_name,
          mp.price as membership_price,
          b.id as benefit_id,
          b.description as benefit_description,
          b.detail as benefit_detail,
          a.id as audience_id,
          a.description as audience_description
      FROM customers c
      INNER JOIN memberships m ON c.id = m.id
      INNER JOIN membershipperiods mp ON m.id = mp.membership_id
      INNER JOIN periods p ON mp.period_id = p.id
      INNER JOIN membershipbenefits mb ON m.id = mb.membership_id 
          AND mp.period_id = mb.period_id
      INNER JOIN benefits b ON mb.benefit_id = b.id
      LEFT JOIN audiences a ON mb.audience_id = a.id
      WHERE 
          m.id IS NOT NULL
          AND p.id IS NOT NULL
      ORDER BY c.name, m.name, b.description;
      ```
      
      ## 🔹 **10. Ver ciudades con cantidad de empresas**
      
      ```sql
      SELECT 
          c.code as city_code,
          c.name as city_name,
          sr.name as state_region,
          co.name as country_name,
          COUNT(comp.id) as total_companies
      FROM citiesormunicialities c
      INNER JOIN stateorregions sr ON c.statereg_id = sr.code
      INNER JOIN countries co ON sr.country_id = co.isocode
      LEFT JOIN companies comp ON c.code = comp.city_id
      GROUP BY c.code, c.name, sr.name, co.name
      ORDER BY total_companies DESC, c.name;
      ```
      
      ## 🔹 **11. Ver encuestas con calificaciones**
      
      ```sql
      SELECT 
          p.id as poll_id,
          p.name as poll_name,
          p.description as poll_description,
          p.isactive as poll_active,
          p.categorypoll_id,
          r.customer_id,
          r.company_id,
          r.daterating as rating_date,
          r.rating as rating_value
      FROM polls p
      INNER JOIN rates r ON p.id = r.poll_id
      ORDER BY p.name, r.daterating DESC;
      ```
      
      ## 🔹 **12. Ver productos evaluados con datos del cliente**
      
      ```sql
      SELECT 
          p.id as product_id,
          p.name as product_name,
          p.detail as product_detail,
          p.price as product_price,
          c.id as customer_id,
          c.name as customer_name,
          c.email as customer_email,
          c.cellphone as customer_phone,
          qp.daterating as evaluation_date,
          qp.rating as product_rating,
          qp.company_id,
          qp.poll_id
      FROM quality_products qp
      INNER JOIN products p ON qp.product_id = p.id
      INNER JOIN customers c ON qp.customer_id = c.id
      ORDER BY qp.daterating DESC, p.name;
      ```
      
      ## 🔹 **13. Ver productos con audiencia de la empresa**
      
      ```sql
      SELECT 
          p.id as product_id,
          p.name as product_name,
          p.detail as product_detail,
          p.price as product_price,
          p.image as product_image,
          cat.description as category_name,
          c.id as company_id,
          c.name as company_name,
          c.email as company_email,
          c.cellphone as company_phone,
          a.id as audience_id,
          a.description as target_audience,
          city.name as company_city,
          sr.name as state_region
      FROM products p
      INNER JOIN categories cat ON p.category_id = cat.id
      INNER JOIN companies c ON p.category_id = c.category_id
      INNER JOIN audiences a ON c.audience_id = a.id
      LEFT JOIN citiesormunicialities city ON c.city_id = city.code
      LEFT JOIN stateorregions sr ON city.statereg_id = sr.code
      ORDER BY cat.description, p.name;
      
      ```
      
      ## 🔹 **14. Ver clientes con sus productos favoritos**
      
      ```sql
      SELECT 
          c.id as customer_id,
          c.name as customer_name,
          c.email as customer_email,
          c.cellphone as customer_phone,
          c.address as customer_address,
          f.id as favorite_id,
          f.company_id as favorite_company_id
      FROM customers c
      INNER JOIN favorites f ON c.id = f.customer_id
      ORDER BY c.name, f.id;
      ```
      
      ## 🔹 **15. Ver planes, periodos, precios y beneficios**
      
      ```sql
      SELECT 
          m.id as membership_id,
          m.name as membership_name,
          m.description as membership_description,
          p.id as period_id,
          p.name as period_name,
          mp.price as membership_price,
          b.id as benefit_id,
          b.description as benefit_description,
          b.detail as benefit_detail,
          a.id as audience_id,
          a.description as target_audience
      FROM memberships m
      INNER JOIN membershipperiods mp ON m.id = mp.membership_id
      INNER JOIN periods p ON mp.period_id = p.id
      LEFT JOIN membershipbenefits mb ON m.id = mb.membership_id 
          AND p.id = mb.period_id
      LEFT JOIN benefits b ON mb.benefit_id = b.id
      LEFT JOIN audiences a ON mb.audience_id = a.id
      ORDER BY m.name, p.name, b.description;
      ```
      
      ## 🔹 **16. Ver combinaciones empresa-producto-cliente calificados**
      
      ```sql
      SELECT 
          qp.company_id,
          comp.name as company_name,
          comp.email as company_email,
          comp.cellphone as company_phone,
          qp.product_id,
          p.name as product_name,
          p.detail as product_detail,
          p.price as product_price,
          qp.customer_id,
          c.name as customer_name,
          c.email as customer_email,
          c.cellphone as customer_phone,
          qp.daterating as rating_date,
          qp.rating as rating_value,
          qp.poll_id
      FROM quality_products qp
      INNER JOIN companies comp ON qp.company_id = comp.id
      INNER JOIN products p ON qp.product_id = p.id
      INNER JOIN customers c ON qp.customer_id = c.id
      ORDER BY qp.daterating DESC, comp.name, p.name, c.name;
      ```
      
      ## 🔹 **17. Comparar favoritos con productos calificados**
      
      ```sql
      SELECT DISTINCT
          p.id as product_id,
          p.name as product_name,
          p.detail as product_detail,
          p.price as product_price,
          p.image as product_image,
          cat.description as category_name,
          comp.id as company_id,
          comp.name as company_name,
          comp.email as company_email,
          qp.rating as my_rating,
          qp.daterating as rating_date,
          f.id as favorite_id
      FROM quality_products qp
      INNER JOIN products p ON qp.product_id = p.id
      INNER JOIN companies comp ON qp.company_id = comp.id
      INNER JOIN categories cat ON p.category_id = cat.id
      INNER JOIN favorites f ON qp.customer_id = f.customer_id 
          AND qp.company_id = f.company_id
      WHERE qp.customer_id = @customer_id
      ORDER BY qp.daterating DESC, p.name;
      ```
      
      ## 🔹 **18. Ver productos ordenados por categoría**
      
      ```sql
      SELECT 
          c.id as category_id,
          c.description as category_name,
          p.id as product_id,
          p.name as product_name,
          p.detail as product_detail,
          p.price as product_price,
          p.image as product_image
      FROM categories c
      INNER JOIN products p ON c.id = p.category_id
      ORDER BY c.description, p.name;
      ```
      
      ## 🔹 **19. Ver beneficios por audiencia, incluso vacíos**
      
      ```sql
      SELECT 
          a.id as audience_id,
          a.description as audience_description,
          b.id as benefit_id,
          b.description as benefit_description,
          b.detail as benefit_detail,
          mb.membership_id,
          mb.period_id
      FROM audiences a
      LEFT JOIN membershipbenefits mb ON a.id = mb.audience_id
      LEFT JOIN benefits b ON mb.benefit_id = b.id
      ORDER BY a.description, b.description;
      ```
      
      ## 🔹 **20. Ver datos cruzados entre calificaciones, encuestas, productos y clientes**
      
      ```sql
      SELECT 
          r.customer_id,
          c.name as customer_name,
          c.email as customer_email,
          c.cellphone as customer_phone,
          r.company_id,
          r.poll_id,
          pol.name as poll_name,
          pol.description as poll_description,
          pol.isactive as poll_active,
          r.daterating as rating_date,
          r.rating as rating_value,
          qp.product_id,
          p.name as product_name,
          p.detail as product_detail,
          p.price as product_price,
          qp.rating as product_rating,
          qp.daterating as product_rating_date
      FROM rates r
      INNER JOIN customers c ON r.customer_id = c.id
      INNER JOIN polls pol ON r.poll_id = pol.id
      INNER JOIN quality_products qp ON r.customer_id = qp.customer_id 
          AND r.company_id = qp.company_id 
          AND r.poll_id = qp.poll_id
      INNER JOIN products p ON qp.product_id = p.id
      ORDER BY r.daterating DESC, c.name, p.name;
      ```
      
      
      
      ### 																									
   
   