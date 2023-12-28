# Shopping Trends Analysis

Understanding customer trends and preferences is crucial for a business to succeed.  In this notebook, we are going to analyze shopping trend data to identify:
  * Who are our customers?
  * What are our top/bottom selling products?
  * Do we experience seasonal trends?


# What is our customer demographic?

Our dataset contains customer age, gender, location and subscription status.<br>
Let's identify what percentage of our customers are male/female and their age group.


```sql
SELECT 
	gender,
	CASE 
    	when age BETWEEN 18 and 29 then '18-29'
        when age BETWEEN 30 and 39 then '30-39'
        when age BETWEEN 40 and 49 then '40-49'
        when age BETWEEN 50 and 59 then '50-59'
        else '60-70'
	END AS age_range,
  	ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM shopping_trends_updated), 2) AS percentage
FROM 
  	shopping_trends_updated
GROUP BY 
  	age_range, gender
ORDER BY 
  	percentage DESC;
```


| Gender | Age Range | Percentage |
|--------|-----------|------------|
| Male   | 18-29     | 15.46      |
| Male   | 60-70     | 13.9       |
| Male   | 50-59     | 13.56      |
| Male   | 30-39     | 12.56      |
| Male   | 40-49     | 12.51      |
| Female | 18-29     | 7.05       |
| Female | 40-49     | 6.44       |
| Female | 60-70     | 6.31       |
| Female | 50-59     | 6.21       |
| Female | 30-39     | 6          |



<br>
Seems majority of our customers are male with < 3% difference between age ranges.<br>
Since we have subscription status, let's identify demographic of our subscribers.<br>
<br>
<br>

```sql
SELECT
  gender,
  SUM(CASE WHEN subscription_status = 'Yes' THEN 1 ELSE 0 END) AS Subscribed,
  SUM(CASE WHEN subscription_status = 'No' THEN 1 ELSE 0 END) AS Not_Subscribed
FROM shopping_trends_updated
GROUP BY gender;
```


| Gender | Subscribed | Not Subscribed |
|--------|------------|-----------------|
| Female | 0          | 1248            |
| Male   | 1053       | 1599            |



<br>
Interesting, it seems none of our female customers are subscribers.<br>
Let's move onto analyzing our location data to identify what region has the most customers.
<br>
<br>
<br>

```sql
SELECT
  region,
  count(*) * 100 / (SELECT count(*) from shopping_trends_updated) as Percentage
from shopping_trends_updated
group by region
order by percentage DESC
```

| Region    | Percentage |
|-----------|------------|
| South     | 28         |
| West      | 26         |
| Midwest   | 24         |
| Northeast | 21         |
<br>
Looks like we have heavier presence in the southern and western regions than we do elsewhere.<br>
Based on our findings, I would suggest we develop plans to become more inclusive to all genders as well as increase our presence in the Northeast region.
<br>
<br>

# What are our top/bottom performing products?

Our dataset contains product name, category and cost.<br>
Let's identify the top and bottom 5 performing products.

```sql
WITH BottomItems AS 
(SELECT
  'Bottom 5' AS Performing_Status,
  Category,
  Item_purchased AS Item,
  SUM(purchase_amount_usd) AS Total_Sales
FROM shopping_trends_updated
GROUP BY item_purchased
ORDER BY Total_Sales ASC
LIMIT 5),

TopItems AS 
(SELECT
  'Top 5' AS Performing_Status,
  Category,
  Item_purchased AS Product,
  SUM(purchase_amount_usd) AS Total_Sales
FROM shopping_trends_updated
GROUP BY item_purchased
ORDER BY Total_Sales DESC
LIMIT 5)

SELECT * from TopItems
UNION
SELECT * from BottomItems
order by Total_Sales DESC


```
| Performing Status | Category    | Product   | Total Sales |
|-------------------|-------------|-----------|-------------|
| Top 5             | Clothing    | Blouse    | 10410       |
| Top 5             | Clothing    | Shirt     | 10332       |
| Top 5             | Clothing    | Dress     | 10320       |
| Top 5             | Clothing    | Pants     | 10090       |
| Top 5             | Accessories | Jewelry   | 10010       |
| Bottom 5          | Clothing    | Hoodie    | 8767        |
| Bottom 5          | Accessories | Backpack  | 8636        |
| Bottom 5          | Footwear    | Sneakers  | 8635        |
| Bottom 5          | Accessories | Gloves    | 8477        |
| Bottom 5          | Clothing    | Jeans     | 7548        |



<br>
Looks like majority our top performing products are in the clothing category.<br>
Let's see how the other categories are performing by finding total sales by category and identifying the top performing product in each category.
<br>
<br>
<br>


```sql
WITH RevenuePerItem AS (
  SELECT
    category as Category,
    item_purchased as Item,
    SUM(purchase_amount_usd) AS Item_Sales,
    RANK() OVER (PARTITION BY category ORDER BY SUM(purchase_amount_usd) DESC) AS rnk
  FROM
    shopping_trends_updated
  GROUP BY
    category, item_purchased
)

SELECT
  rpi.category,
  SUM(purchase_amount_usd) 	AS Category_Sales,
  rpi.item                      AS Top_Selling_Item,
  rpi.Item_Sales                AS Top_Selling_Item_Sales
FROM
  RevenuePerItem rpi
JOIN
  shopping_trends_updated stu ON stu.category = rpi.category
WHERE
  rpi.rnk = 1
group by rpi.category
ORDER BY
  Category_Sales DESC
```


| Category     | Category Sales | Top Selling Product | Top Selling Product Sales |
|--------------|----------------|---------------------|---------------------------|
| Clothing     | 104264         | Blouse              | 10410                     |
| Accessories  | 74200          | Jewelry             | 10010                     |
| Footwear     | 36093          | Shoes               | 9240                      |
| Outerwear    | 18524          | Coat                | 9275                      |

<br>
It seems Outerwear and Footwear have significantly lower sales than Clothing and Accessories although the top selling products for each are performing rather well.<br>
Let's see how many unique products we have available in each category and identify any correlation.
<br>
<br>
<br>


```sql
SELECT
  category,
  SUM(purchase_amount_usd)	AS Category_Sales,
  count(DISTINCT(item_purchased)) as Unique_Items
from shopping_trends_updated
group by category
order by 2 DESC;
```


| Category     | Category Sales | Unique Items |
|--------------|----------------|--------------|
| Clothing     | 104264         | 11           |
| Accessories  | 74200          | 8            |
| Footwear     | 36093          | 4            |
| Outerwear    | 18524          | 2            |


We can definitely see some correlation here between products available and total category sales.<br>
I would suggest we look into increasing our available products for Outerwear and Footwear.


# Do we experience seasonal trends?

Our dataset also contains size, color, season and if a purchase was made due to a promotion.<br>
Let's see if we experience a busy season.
<br>
```sql
SELECT
  Season,
  SUM(purchase_amount_usd) AS Total_Sales
FROM shopping_trends_updated
GROUP BY Season
order by 2 DESC
```
| Season | Total Sales            |
|--------|------------------------|
| Fall   | 60018                  |
| Spring | 58679                  |
| Winter | 58607                  |
| Summer | 55777                  |

<br>
Looks like Fall is our busiest season while we tend to slow down during Summer.<br>
Let's look into Summer and identify our bottom 10 performing products by total purchases.
<br>
<br>
<br>


```sql
SELECT
  Season,
  item_purchased AS Item,
  COUNT(*) AS Purchase_Count,
  SUM(purchase_amount_usd) AS Total_Sales
FROM shopping_trends_updated
WHERE season = 'Summer'
GROUP BY season, item
order by 3 ASC
LIMIT 10
```

| Season | Item      | Purchase Count | Total Sales |
|--------|-----------|-----------------|-------------|
| Summer | Skirt     | 28              | 1589        |
| Summer | Sweater   | 28              | 1518        |
| Summer | Gloves    | 29              | 1733        |
| Summer | T-shirt   | 30              | 1640        |
| Summer | Hoodie    | 31              | 1653        |
| Summer | Jeans     | 31              | 1842        |
| Summer | Jacket    | 33              | 1757        |
| Summer | Handbag   | 35              | 1990        |
| Summer | Sneakers  | 36              | 2114        |
| Summer | Hat       | 37              | 2233        |


<br>
Hm, makes sense that sweaters, gloves and hoodies would be underperforming during warmer weather.<br>
Although skirts, handbags and hats shouldn't be effected.<br>
Let's identify top selling colors so we can adjust our stock.
<br>
<br>
<br>


```sql
WITH SeasonalRankings AS (
  SELECT
    Season,
    CASE
      WHEN color IN ('Red', 'Orange', 'Maroon') THEN 'Warm_Red_Tones'
      WHEN color IN ('Gold', 'Peach') THEN 'Warm_Golden_Tones'
      WHEN color IN ('Blue', 'Cyan', 'Indigo') THEN 'Cool_Blue_Tones'
      WHEN color IN ('Teal', 'Turquoise') THEN 'Cool_Teal_Tones'
      WHEN color IN ('Beige', 'Brown', 'Charcoal') THEN 'Neutral_Brown_Tones'
      WHEN color IN ('Gray', 'Olive', 'Silver') THEN 'Neutral_Gray_Tones'
      WHEN color IN ('Black', 'Charcoal') THEN 'Neutral_Black_Tones'
      WHEN color IN ('Green') THEN 'Various_Green'
      WHEN color IN ('Lavender', 'Magenta', 'Pink', 'Purple', 'Violet') THEN 'Various_Purple_Tones'
      WHEN color IN ('White') THEN 'Neutral_White'
      WHEN color IN ('Yellow') THEN 'Warm_Yellow'
      ELSE 'Uncategorized'
    END AS Color_Group,
    COUNT(*) AS Purchase_Count,
    SUM(purchase_amount_usd) AS Total_Sales,
    ROW_NUMBER() OVER (PARTITION BY season ORDER BY SUM(purchase_amount_usd) DESC) AS rank
  FROM shopping_trends_updated
  GROUP BY season, color_group
)

SELECT Season, Color_Group, Purchase_Count, Total_Sales
FROM SeasonalRankings
WHERE rank <= 3;
```


| Season | Color Group             | Purchase Count | Total Sales |
|--------|-------------------------|-----------------|-------------|
| Fall   | Various Purple Tones    | 204             | 12505       |
| Fall   | Neutral Gray Tones      | 130             | 8136        |
| Fall   | Warm Red Tones          | 127             | 7818        |
| Spring | Various Purple Tones    | 201             | 11790       |
| Spring | Neutral Gray Tones      | 137             | 7746        |
| Spring | Neutral Brown Tones     | 114             | 6833        |
| Summer | Various Purple Tones    | 181             | 10718       |
| Summer | Neutral Gray Tones      | 134             | 7731        |
| Summer | Cool Blue Tones         | 120             | 6794        |
| Winter | Various Purple Tones    | 183             | 10940       |
| Winter | Neutral Brown Tones     | 120             | 7322        |
| Winter | Cool Blue Tones         | 115             | 7033        |

<br>
It seems that various purple tones and neutral gray tones are our top performing color for majority of the year.<br>
Our dataset also includes if a promotion code was used.  Let's check our bottom 10 performing products and see what percentages of purchased were from a promotion.
<br>
<br>
<br>


```sql
WITH BottomPerforming AS (
  SELECT
    item_purchased AS Item,
    COUNT(*) AS Purchase_Count,
    SUM(purchase_amount_usd) AS Total_Sales
  FROM shopping_trends_updated
  WHERE season = 'Summer'
  GROUP BY season, item
  ORDER BY 2 ASC
  LIMIT 10
)

SELECT
  season,
  item_purchased,
  SUM(CASE WHEN promo_code_used = 'Yes' THEN 1 ELSE 0 END) AS 'Promo_Used',
  SUM(CASE WHEN promo_code_used = 'No' THEN 1 ELSE 0 END) AS 'Promo_Not_Used',
  ROUND(SUM(CASE WHEN promo_code_used = 'Yes' THEN 1 ELSE 0 END) * 100.0 / COUNT(*), 2) AS 'Percentage_Promo_Used'
FROM shopping_trends_updated
WHERE season = 'Summer'
  AND item_purchased IN (SELECT Item FROM BottomPerforming)
GROUP BY season, item_purchased
ORDER BY 5 ASC;

```
| Season | Item Purchased | Promo Used | Promo Not Used | Percentage Promo Used |
|--------|----------------|------------|----------------|-----------------------|
| Summer | T-shirt        | 11         | 19             | 36.67                 |
| Summer | Jacket         | 13         | 20             | 39.39                 |
| Summer | Handbag        | 14         | 21             | 40                    |
| Summer | Gloves         | 13         | 16             | 44.83                 |
| Summer | Jeans          | 14         | 17             | 45.16                 |
| Summer | Skirt          | 13         | 15             | 46.43                 |
| Summer | Sneakers       | 17         | 19             | 47.22                 |
| Summer | Hoodie         | 15         | 16             | 48.39                 |
| Summer | Hat            | 20         | 17             | 54.05                 |
| Summer | Sweater        | 19         | 9              | 67.86                 |
<br>
It looks like we could be experiencing poor T-shirt sales due to lack of promotions.<br>
We should also evaluate our Summer inventory to ensure we are pushing seasonal colors.

<br>
<br>
<br>

# Conclusion

  - Who are our customers?
    - 68% of our customers and 100% of our subscribers are male.
  - What are our top/bottom selling products?
    - Clothing is our highest performing category while Outerwear is our least.  This could be due to Clothing having 11 distinct products while Outerwear only has 2.
  - Do we experience seasonal trends?
    - Fall is our busiest season while Summer is our least busy.  This could be due to incorrect color palette choices or Summer related products not being promoted efficiently.


