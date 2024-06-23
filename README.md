#### Table of Contents
* [Dashboarding](#dashboarding)
  - [Summary of dashboard questions](#summary-of-dashboard-questions)
  - [Dashboard presentation](##dashboard-presentation)
  - [Approach Explained in Details](#approach-explained-in-details)
      - [Understanding the sample data](#understanding-the-sample-data)
      - [Ingesting the sample data](#ingesting-the-sample-data)
      - [Data transformation](#data-transformation)
      - [Connect data to BI tool with customized SQL query](#connect-data-to-BI-tool-with-customized-sql-query)
      - [Dashboarding in Tableau](#dashboarding-in-tableau)
      - [Publish dashboard to public](#publish-dashboard-to-public)
* [Additional Question1](#additional-question1)
* [Additional Question2](#additional-question2)

# Dashboarding (view Tableau dashboard)
## Summary of dashboard questions
* [ ]  How many customers visited the "Restaurant at the end of the universe"? 689 
* [ ]  How much money did the "Restaurant at the end of the universe" make? N/A
&nbsp;

_Notice: This question could not be answered due to the missing value of menu_price in the given data.csv file. However, in order to create a complete dashboarding experience, test value of menu price has been added to the data.csv as an additional column (menu_price). So that total could be calculated by the sum of menu price._
* [ ]  What was the most popular dish at each restaurant?
* bean-juice-stand: honey
* johnnys-cashew-stand: juice
* the-ice-cream-parlor: bean
* the-restaurant-at-the-end-of-the-universe: cheese

* [ ]  What was the most profitable dish at each restaurant? N/A
&nbsp;

Notice: This question could not be answered due to the missing value of menu_price in the given data.csv file. However, in order to create a complete dashboarding experience, test value of menu price has been added to the data.csv as an additional column (menu_price). So that profitability could be calculated (Profitability = Menu Price - Food Cost). 

* [ ]  Who visited each store the most, and who visited the most stores?
Michael; Michael

## [Dashboard Presentation](https://public.tableau.com/app/profile/jasmine.shen/viz/dataoperationsproject/Summary?publish=yes)
The dashboard is composed of 3 tabs:
* Summary page: This tab gives an high level overview of the dashboard: customer visits view and dish sales view; answers to the dashboard questions.
<img width="1512" alt="tableau-v1" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/aca71473-0723-413c-89ce-286bb5d9ed11">


* Customer visits: This tab provides a more detailed view of customer and visits related questions.
<img width="1512" alt="tableau-v2" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/d9f373d1-c043-4795-8273-4f7252bc7b3f">

* Dish sales: This tab provides a more detailed view of revenue, dish popularity and dish profitability related questions.
<img width="1512" alt="tableau-v3" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/207305f1-a2d3-480f-a550-0ca51ae4e96a">

## Approach Explained in Details
### Understanding the sample data 
I first loaded the data into notebook (Collab) and conducted some checks to gain a basic understanding of the data quality and attributes. 
```python
import pandas as pd
url = "https://raw.githubusercontent.com/TableCheck-Labs/tablecheck-data-operations-take-home/main/data/data.csv"
data = pd.read_csv(url)

data.head()

# check: how big the file is? Not big.
total_rows = len(data)
print(f"Total number of rows: {total_rows}")

# check: the data type of each columns
data_types = data.dtypes
print("Data types for each column:")
print(data_types)

# check: how many distinct restaurants?
distinct_restaurant_names = data['restaurant_names'].nunique()
print(f"Distinct restaurant name count: {distinct_restaurant_names}")

# list: all the distinct restaurant names: no caps/lower case issues here
restaurant_names_list = data['restaurant_names'].unique()
print(restaurant_names_list)

# check: how many distinct food names/dishes?
distinct_food_names = data['food_names'].nunique()
print(f"Distinct food name count: {distinct_food_names}")

# list: all the distinct food names: no caps/lower case issues here
food_name_list = data['food_names'].unique()
print(food_name_list)

# check: does each restaurant sell different dishes? do restaurants sell some of the same dishes? Each restaurant sells exactly 34 dishes..
dishes_restaurants = {}
for i in range(len(data)):
  dish = data.loc[i, 'food_names']
  restaurant = data.loc[i, 'restaurant_names']
  if dish not in dishes_restaurants:
    dishes_restaurants[dish] = [restaurant]
  else:
    if restaurant not in dishes_restaurants[dish]:
      dishes_restaurants[dish].append(restaurant)

multi_restaurant_dishes = {dish: restaurants for dish, restaurants in dishes_restaurants.items() if len(restaurants) > 1}
multi_restaurant_dishes_count = len(multi_restaurant_dishes)

print(f"Total number of dishes sold in more than 1 restaurant: {multi_restaurant_dishes_count}")

for dish, restaurants in multi_restaurant_dishes.items():
  print(f"{dish}: {restaurants}")

# check: how many distinct customers?
distinct_customer_name=data['first_name'].nunique()
print(f"Distinct customer name count: {distinct_customer_name}")

# list: all the distinct customer first names: no caps/lower case issues here
first_name_list = data['first_name'].unique()
print(first_name_list)

# check: for the same dish in each restaurant, does the food cost always the same? No.
grouped_data = data.groupby(['restaurant_names', 'food_names'])
is_food_cost_same = grouped_data['food_cost'].apply(lambda x: x.nunique() == 1)
print(is_food_cost_same)
```


### Ingesting the sample data
In this assignment, snowflake is chosen - simply because it's so easy to load an external table manually. Also, initially, I wanted to test the dashboard feature within snowflake - but eventually decided to opt for Tableau due to better visualizations.
1. Creating database and schema
```SQL
use role accountadmin;

-- create a database
create database tablecheck_db
comment='this database is created for tablecheck assignment';
show database;

-- create a schema
create schema github
comment='this schema contains data from github under tablecheck_db';
show schemas;
```
<img width="1429" alt="SF_create_db_and_schema" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/e081c969-8034-483b-a045-236d6e71591e">

2. Manually loading the data into the above database & schema 
<img width="1246" alt="SF_load_data1" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/25d572e3-7e12-41d2-b2f6-dc060c7ae918">
<img width="1252" alt="SF_load_data2" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/130bd8bf-6e55-4aed-8c8c-feeaac5dc00d">

### Data transformation
Some of the dashboard questions (eg. total revenue and dish profitability) could not be answered due to missing attribute/field Menu price. In order to create a better dashboarding experience, I decided to add some logical dummy data into the existing table (data.csv file) and save it as another table named "data_incl_menu_price".
&nbsp;

The basic approach is to: 1) First generate random food cost ration between 20% -40%: most restaurants' food cost percentage will fall in this range, assuming that the restaurants in the dataset are having a relatively healthy business performance. 2)Based on the food cost percentage, menu price could be calculated by "food_cost/food_cost_percentage".
&nbsp;

Here's the python file to create the logical dummy data 
```python
# The Snowpark package is required for Python Worksheets. 
# You can add more packages by selecting them using the Packages control and then importing them.

import snowflake.snowpark as snowpark
from snowflake.snowpark.functions import col, round, udf
from snowflake.snowpark.types import FloatType
import random

def main(session: snowpark.Session): 
    # Your code goes here, inside the "main" handler.
    tableName = 'tablecheck_db.github.data'
    
    # 1. Generate column food_cost_percentage
    dataframe = session.table(tableName)
    
    # Define UDF with specified return type
    @udf(return_type=FloatType())
    def random_percentage():
        return random.uniform(0.2, 0.4)
    
    dataframe = dataframe.with_column('food_cost_percentage', round(random_percentage(), 2))
    
    # 2. Calculate menu_price
    dataframe = dataframe.with_column('menu_price', round(col('food_cost') / col('food_cost_percentage'), 2))

    # 3. Save the DataFrame to a new table
    dataframe.write.mode("overwrite").save_as_table("data_incl_menu_price")

    # Print a sample of the dataframe to standard output.
    dataframe.show()

    # Return value will appear in the Results tab.
    return dataframe 
```
<img width="1421" alt="SF_add_menu_price_column" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/d4ce691f-610d-4e37-9b32-1a0039362c33">

### Connect data to BI tool with customized SQL query
As mentioned earlier, eventually I decided to opt for better visualization instead of ease of implementation - therefore, Tableau was chosen. So we need to connect snowflake data to Tableau. Despite that 150k rows is not much data to Tableau, I still chose to connect Tableau with 2 customized SQL queries for 2 reasons:
* Lighter: 136 rows for dish sales table, and 2.8k rows for customer visit table.
&nbsp;
<img width="1511" alt="tableau_connect_to_data" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/9d28441a-b0b9-4ea8-a2d4-0c65b0bd0759">

dish_sales table:
```SQL
SELECT
restaurant_names, food_names,sum(menu_price) as total_revenue, sum(food_cost) as food_cost, count(food_names) as dish_count
FROM 
tablecheck_db.github.data_incl_menu_price
GROUP BY restaurant_names,food_names
```

customer_visit table:
```SQL
SELECT
restaurant_names, first_name,count(first_name) as no_of_visit
FROM 
tablecheck_db.github.data_incl_menu_price
GROUP BY restaurant_names,first_name;
```

* Clearer business logics: The dashboard questions can be clearly categorized into 2 types - customer visits and dish sales analytics. Even though the dataset is small and we don't need to create intermediary tables/data marts according to different domains, but personally, I feel it's more organized..

### Dashboarding in Tableau
Nothing special and not too much effort needed in this part - basic visualizations are used because I believe in simple is more. I tried to pay attention to the details such as:
* Highlighter: Highlight the bars of "the-restaurant-at-the-end-of-the-universe"
* Color scheme: Unfortunately it seems that Tableau only supports default color palette when it comes to the setting for multiple color in the bar chart. I could only pick a purple-ish color.
* Navigations
* Dynamic TopN filters

### Publish dashboard to public
Once the dashboard is done, I published the Tableau workbook to Tableau Public - where users can publicly share data visualizations. In case some other candidate are doing the same assignment - I turned off the "Show Viz on Profile" button, which means access will only be granted with the dashoard ur.
<img width="423" alt="Screenshot 2024-06-24 at 05 01 52" src="https://github.com/jasmine-shen/data-operations-project-tablecheck/assets/173486330/da8b8067-21c8-40c3-8109-e8bd94aeac48">

# Additional Question1
### How would you build this differently if the data was being streamed from Kafka?
1. Tool selection: Kafka needs to be installed at cloud based service. Also, tools that supports to read data from Kafka - Luckily snowflake has "Kafka connector", but their might be other better options that handles ingestions in a more efficient way. 
2. Data importing methods: Manual will fail - need a distributed streaming platform, eg. Flink/Apache. Snowflake has snowpipe streaming thatloads data through streming API. 
3. Data processing: ETL is needed because the number of records will hike. Also in reality, the business model is way more complex. 
4. Data storage: More cost-efficient data storage methods could be considered. 
5. Visualizations: BI tools that are specialized in real-time streaming analytics could be considered - eg. Power BI. 

# Additional Question2
### How would you improve the deployment of this system?
I personally would not call my output of this assignment "deployment" because there's no test environment. Therefore, the biggest improvement needed is to have proper tests in place:
* Committing changes in the database
* Dashboarding environment: separate stage vs.dev

Also considering the nature of the Tablecheck data, nature of the business, standardized flow of development cycle, as well as the hurdles that I encountered when doing this assignment, below areas could be improved/implemented:
1. Business metrics systems
2. Automated data quality checks
3. Standardized ETL process/ dedicated reporting database 
4. Automated dashboard testing process
5. Personal improvement: documentation automation. 

