# Python-sales
This code connects to the sql data base and delivers the sales data for top performing products
Author - Mohammad Usman Adhami
import pandas as pd
import pyathena
import time

conn = pyathena.connect(
    aws_access_key_id='',
    aws_secret_access_key='',
    work_group='',
    region_name='',
    endpoint_url='https
)

# Record the start time
start_time = time.time()


# Define the first query for the original data
query1 = """
SELECT
  ADK_EXT_AS_ID,
  adk_product_net_sales_bmf,
  std.ap_id,
  std.brand_name,
  ADK_DATE
FROM datamarts_openspace.PRC_AS_DAILY_KPIS kpi
JOIN datamarts_openspace.prc_std_query_all_daily std
ON kpi.ADK_EXT_AS_ID = std.as_id
WHERE
  ADK_DATE BETWEEN date '2023-06-01' AND date '2023-08-30'
  and std.site_id in (32)
"""

# Execute the first query for the original data and read the result
df1 = pd.read_sql_query(query1, conn)

# Group by ADK_EXT_AS_ID, month, and year to calculate the monthly net sales
df1['ADK_DATE'] = pd.to_datetime(df1['ADK_DATE'])
df1['Year'] = df1['ADK_DATE'].dt.year
df1['Month'] = df1['ADK_DATE'].dt.month
monthly_data = df1.groupby(['ADK_EXT_AS_ID', 'Year', 'Month'])['adk_product_net_sales_bmf'].sum().reset_index()

# Define the second query for the top sellers
query2 = """
WITH SalesSum AS (
  SELECT
    std.brand_name,
    std.pg_l3,
    SUM(kpi.adk_product_net_sales_bmf) AS total_sales
  FROM
    datamarts_openspace.PRC_AS_DAILY_KPIS kpi
    JOIN datamarts_openspace.prc_std_query_all_daily std
      ON kpi.ADK_EXT_AS_ID = std.as_id
  WHERE
    kpi.ADK_DATE BETWEEN from_iso8601_timestamp('2022-10-24T00:00:00.000Z') AND current_date
    AND std.site_id = 32
  GROUP BY
    std.brand_name,
    std.pg_l3
)
SELECT
  brand_name,
  pg_l3 AS product_category,
  total_sales
FROM SalesSum
ORDER BY total_sales DESC;
"""

# Execute the second query for the top sellers and read the result
df2 = pd.read_sql_query(query2, conn)

# Create an Excel writer
with pd.ExcelWriter('sales_data.xlsx', engine='xlsxwriter') as writer:
    # Write the original data to a sheet named 'Original Data'
    df1.to_excel(writer, sheet_name='Original Data', index=False)

    # Write the monthly data to a sheet named 'Monthly Sales Data'
    monthly_data.to_excel(writer, sheet_name='Monthly Sales Data', index=False)

    # Write the top sellers data to a sheet named 'Top Sellers'
    df2.to_excel(writer, sheet_name='Top Sellers', index=False)

# Print a message indicating the export is complete
print("Data exported to 'sales_data.xlsx'")

# Record the end time
end_time = time.time()

# Calculate and print the execution time
execution_time = end_time - start_time
print(f"Query execution time: {execution_time:.2f} seconds")
