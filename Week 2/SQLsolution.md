## Schema of the dataset


    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');

---
## A few records from each table


    select * from sales limit 5;

| customer_id | order_date               | product_id |
| ----------- | ------------------------ | ---------- |
| A           | 2021-01-01T00:00:00.000Z | 1          |
| A           | 2021-01-01T00:00:00.000Z | 2          |
| A           | 2021-01-07T00:00:00.000Z | 2          |
| A           | 2021-01-10T00:00:00.000Z | 3          |
| A           | 2021-01-11T00:00:00.000Z | 3          |

---


    select * from menu limit 5;

| product_id | product_name | price |
| ---------- | ------------ | ----- |
| 1          | sushi        | 10    |
| 2          | curry        | 15    |
| 3          | ramen        | 12    |

---


    select * from members limit 5;

| customer_id | join_date                |
| ----------- | ------------------------ |
| A           | 2021-01-07T00:00:00.000Z |
| B           | 2021-01-09T00:00:00.000Z |

---
## Solving the questions

1. What is the total amount each customer spent at the restaurant?

#

    SELECT
      sales.customer_id,
      SUM(menu.price) AS total_sales
    FROM sales
    INNER JOIN menu on sales.product_id = menu.product_id
    GROUP BY
      sales.customer_id;

| customer_id | total_sales |
| ----------- | ----------- |
| B           | 74          |
| C           | 36          |
| A           | 76          |

---
2. How many days has each customer visited the restaurant?

#

    SELECT
      customer_id,
      COUNT(distinct order_date) as number_of_days
    FROM sales
    GROUP BY 1;

| customer_id | number_of_days |
| ----------- | -------------- |
| A           | 4              |
| B           | 6              |
| C           | 2              |

---
3. What was the first item from the menu purchased by each customer?
#

    WITH ordered AS (
      SELECT
        sales.customer_id,
        
        RANK() OVER (
          PARTITION BY sales.customer_id
          ORDER BY sales.order_date
        ) AS order_rank,
        menu.product_name
      FROM sales
      INNER JOIN menu
        ON sales.product_id = menu.product_id
    )
    SELECT DISTINCT
      customer_id,
      product_name
    FROM ordered
    
    WHERE order_rank = 1;

| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | curry        |
| C           | ramen        |

---

4. What is the most purchased item on the menu and how many times was it purchased by all customers?
#

    SELECT
      menu.product_name,
      COUNT(*) AS total_purchases
    FROM sales
    INNER JOIN menu
      ON sales.product_id = menu.product_id
    GROUP BY menu.product_name
    ORDER BY total_purchases DESC
    LIMIT 1;

| product_name | total_purchases |
| ------------ | --------------- |
| ramen        | 8               |

---
5. Which item was the most popular for each customer?
#

    WITH customer_cte AS (
      SELECT
        sales.customer_id,
        menu.product_name,
        COUNT(*) AS item_quantity,
        RANK() OVER (
          PARTITION BY sales.customer_id
          ORDER BY COUNT(sales.product_id) desc
        ) AS item_rank
      FROM sales
      INNER JOIN menu
      	ON sales.product_id = menu.product_id
      GROUP BY
        sales.customer_id,menu.product_name
    )
    SELECT
      customer_id,
      product_name,
      item_quantity
    FROM customer_cte
    WHERE item_rank = 1;

| customer_id | product_name | item_quantity |
| ----------- | ------------ | ------------- |
| A           | ramen        | 3             |
| B           | ramen        | 2             |
| B           | curry        | 2             |
| B           | sushi        | 2             |
| C           | ramen        | 3             |

---
6. Which item was purchased first by the customer after they became a member?
#

    WITH member_sales_cte AS (
      SELECT
        sales.customer_id,
        sales.order_date,
        menu.product_name,
        
        RANK() OVER (
          PARTITION BY sales.customer_id
          ORDER BY sales.order_date
        ) AS order_rank
      FROM sales
      INNER JOIN menu
        ON sales.product_id = menu.product_id
      INNER JOIN members
        ON sales.customer_id = members.customer_id
      WHERE
        sales.order_date >= members.join_date::DATE
    )
    SELECT DISTINCT
      customer_id,
      order_date,
      product_name
    FROM member_sales_cte
    WHERE order_rank = 1;

| customer_id | order_date               | product_name |
| ----------- | ------------------------ | ------------ |
| A           | 2021-01-07T00:00:00.000Z | curry        |
| B           | 2021-01-11T00:00:00.000Z | sushi        |

---
7. Which item was purchased just before the customer became a member?
#

    WITH member_sales AS (
      SELECT
        sales.customer_id,
        sales.order_date,
        menu.product_name,
        
        RANK() OVER (
          PARTITION BY sales.customer_id
          ORDER BY sales.order_date DESC
        ) AS order_rank
      FROM sales
      INNER JOIN menu
        ON sales.product_id = menu.product_id
      INNER JOIN members
        ON sales.customer_id = members.customer_id
      WHERE
        sales.order_date < members.join_date::DATE
    )
    SELECT
      customer_id,
      order_date,
      product_name
    FROM member_sales
    WHERE order_rank = 1;

| customer_id | order_date               | product_name |
| ----------- | ------------------------ | ------------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        |
| A           | 2021-01-01T00:00:00.000Z | curry        |
| B           | 2021-01-04T00:00:00.000Z | sushi        |

---
8. What is the total items and amount spent for each member before they became a member?
#

    SELECT
      sales.customer_id,
      COUNT(DISTINCT sales.product_id) AS unique_menu_items,
      SUM(menu.price) AS total_spend
    FROM sales
    INNER JOIN menu
      ON sales.product_id = menu.product_id
    INNER JOIN members
      ON sales.customer_id = members.customer_id
    WHERE
      sales.order_date < members.join_date::DATE
    GROUP BY sales.customer_id;

| customer_id | unique_menu_items | total_spend |
| ----------- | ----------------- | ----------- |
| A           | 2                 | 25          |
| B           | 2                 | 40          |

---
9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
#

    SELECT
      sales.customer_id,
      SUM (
        CASE
          WHEN menu.product_name = 'sushi' THEN 2 * 10 * menu.price
          ELSE 10 * menu.price
        END
      ) AS points
    FROM sales
    LEFT JOIN menu
      ON sales.product_id = menu.product_id
    GROUP BY customer_id
    ORDER BY points DESC;

| customer_id | points |
| ----------- | ------ |
| B           | 940    |
| A           | 860    |
| C           | 360    |

---
10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?
#

    SELECT
      sales.customer_id,
      SUM(
        CASE
        
          WHEN sales.order_date BETWEEN
            members.join_date::DATE AND (members.join_date::DATE+6)
            THEN 2 * 10 * menu.price
          WHEN menu.product_name = 'sushi' THEN 2* 10 * menu.price
          ELSE  10 * menu.price
        END
      ) AS points
    FROM sales
    INNER JOIN menu
      ON sales.product_id = menu.product_id
    INNER JOIN members
      ON sales.customer_id = members.customer_id
    WHERE sales.order_date <= '2021-01-31'::DATE
    GROUP BY sales.customer_id
    ORDER BY points;

| customer_id | points |
| ----------- | ------ |
| B           | 820    |
| A           | 1370   |

---
11. Recreate that table
#

    SELECT
      sales.customer_id,
      sales.order_date,
      menu.product_name,
      menu.price,
      
      CASE WHEN sales.order_date >= members.join_date::DATE THEN 'Y'
        ELSE 'N'
      END AS member
    FROM sales
    INNER JOIN menu
      ON sales.product_id = menu.product_id
    
    INNER JOIN members
      ON sales.customer_id = members.customer_id
    ORDER BY
      sales.customer_id,
      sales.order_date;

| customer_id | order_date               | product_name | price | member |
| ----------- | ------------------------ | ------------ | ----- | ------ |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10    | N      |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15    | Y      |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15    | N      |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10    | N      |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10    | Y      |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12    | Y      |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12    | Y      |

---
12. Rank products (only after they become members)
#

    WITH joint_sales AS (
    SELECT
      sales.customer_id,
      sales.order_date,
      menu.product_name,
      menu.price,
      
      CASE
        WHEN sales.order_date >= members.join_date::DATE THEN 'Y'
        ELSE 'N'
      END as member
    FROM sales
    INNER JOIN menu
      ON sales.product_id = menu.product_id
    LEFT JOIN members
      ON sales.customer_id = members.customer_id
    )
    SELECT
      customer_id,
      order_date,
      product_name,
      price
      member,
      
      
      CASE
        WHEN member = 'Y' then DENSE_RANK() OVER (
          PARTITION BY customer_id,member
          ORDER BY order_date::DATE)
        ELSE null
       END  as rankingg
    FROM joint_sales;

| customer_id | order_date               | product_name | member | rankingg |
| ----------- | ------------------------ | ------------ | ------ | -------- |
| A           | 2021-01-01T00:00:00.000Z | sushi        | 10     |          |
| A           | 2021-01-01T00:00:00.000Z | curry        | 15     |          |
| A           | 2021-01-07T00:00:00.000Z | curry        | 15     | 1        |
| A           | 2021-01-10T00:00:00.000Z | ramen        | 12     | 2        |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12     | 3        |
| A           | 2021-01-11T00:00:00.000Z | ramen        | 12     | 3        |
| B           | 2021-01-01T00:00:00.000Z | curry        | 15     |          |
| B           | 2021-01-02T00:00:00.000Z | curry        | 15     |          |
| B           | 2021-01-04T00:00:00.000Z | sushi        | 10     |          |
| B           | 2021-01-11T00:00:00.000Z | sushi        | 10     | 1        |
| B           | 2021-01-16T00:00:00.000Z | ramen        | 12     | 2        |
| B           | 2021-02-01T00:00:00.000Z | ramen        | 12     | 3        |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12     |          |
| C           | 2021-01-01T00:00:00.000Z | ramen        | 12     |          |
| C           | 2021-01-07T00:00:00.000Z | ramen        | 12     |          |

---

