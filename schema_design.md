# Schema Design Best Practices

## Priciples

* Choose the right data types: Choose the most appropriate data types for columns based on the data size, range, and precision. For example, use INT for integer values, DECIMAL for financial data, and VARCHAR for variable-length text.

* Choose an appropriate primary key: The primary key should be unique and not null, and choose the length of the primary key carefully to avoid wasting storage space.

* Use secondary keys wisely: Add secondary indexes on columns used frequently in WHERE clauses and JOIN operations. The order of the columns in the index is also important, so choose the order wisely based on the query patterns.

* Use composed indexes to optimize queries: A composed index can be used to improve the performance of queries that filter on multiple columns.

* Use covering indexes: A covering index includes all the columns that are needed to satisfy a query, so the query can be fulfilled by scanning the index only.

* Normalize data to reduce redundancy: Avoid storing the same data in multiple places, which can lead to inconsistency and extra storage usage.


```SQL
CREATE TABLE orders (
  order_id INT NOT NULL AUTO_INCREMENT,
  customer_id INT NOT NULL,
  order_date DATE NOT NULL,
  total DECIMAL(10, 2) NOT NULL,
  PRIMARY KEY (order_id),
  INDEX idx_customer_id (customer_id),
  INDEX idx_order_date_customer_id (order_date, customer_id),
  INDEX idx_customer_id_order_date (customer_id, order_date),
  INDEX idx_total (total)
) ENGINE=InnoDB;

-- INSERT example
INSERT INTO orders (customer_id, order_date, total)
VALUES
(1, '2023-02-17', 10.00),
(2, '2023-02-16', 20.00),
(3, '2023-02-15', 30.00);

-- SELECT example with a covering index
SELECT order_id, total
FROM orders
WHERE customer_id = 2;

-- SELECT example with a composed index
SELECT order_id, total
FROM orders
WHERE customer_id = 2 AND order_date = '2023-02-16';
```

## Column Type
In TiDB, there are several data types to choose from, including INT, VARCHAR, TEXT, FLOAT, DOUBLE, BOOLEAN, DATE, DATETIME, and more. Here are a few considerations to keep in mind when choosing data types:

* Choose the smallest data type that can store your data: Using smaller data types can help reduce the amount of storage required for your database. For example, use TINYINT instead of INT if your column only needs to store values between -128 and 127.

* Use variable-length data types when appropriate: Variable-length data types like VARCHAR and TEXT can be more efficient for storing strings than fixed-length data types like CHAR. This is because variable-length data types only use the storage required for the actual data being stored, while fixed-length data types always use the maximum amount of storage.

* Consider the maximum length of your data: When choosing a data type, consider the maximum length of the data being stored. For example, if you have a column that will never need to store more than 10 characters, you can use a VARCHAR(10) data type instead of a larger data type like VARCHAR(255).

* Be aware of the impact on query performance: The data types you choose can also impact query performance. For example, using VARCHAR instead of CHAR can improve storage efficiency, but it can also impact query performance because variable-length data types require additional overhead to store and retrieve.

* Consider the character set: If you're storing string data, be aware of the character set being used. For example, using the UTF-8 character set can improve compatibility with different languages and scripts, but it can also impact storage requirements.

Overall, choosing the right data types for your columns is an important aspect of schema design. By choosing the appropriate data types, you can improve storage efficiency, optimize query performance, and ensure that your database can handle the workload of your application.

### Floating-point Number Types
When it comes to choosing the right data type for floating-point numbers, there are a few considerations to keep in mind.

* Precision: The FLOAT, DOUBLE, and DECIMAL data types all offer different levels of precision. FLOAT is a 32-bit floating-point number with a precision of about 7 decimal places, DOUBLE is a 64-bit floating-point number with a precision of about 15 decimal places, and DECIMAL is a fixed-point number with a user-defined precision.

* Storage space: FLOAT requires 4 bytes of storage space, DOUBLE requires 8 bytes, and DECIMAL requires variable amounts of storage space depending on the precision defined.

* Range of values: FLOAT and DOUBLE have a wider range of values than DECIMAL, which is limited by the user-defined precision.

In general, the best practice for choosing a floating-point data type is to use DECIMAL when precision is important, such as when dealing with financial data. For other applications, DOUBLE is often sufficient and provides a good balance between precision and storage space. FLOAT is generally only used when storage space is at a premium or precision is not important.

It's also important to consider the impact of floating-point rounding errors on your application. Depending on the specific use case, it may be necessary to use a different data type or perform calculations in a different way to avoid rounding errors.

In summary, the best practice for choosing a floating-point data type depends on the specific requirements of your application. Consider factors such as precision, storage space, and range of values when making your decision, and be aware of the potential for rounding errors when using floating-point numbers.

## Index

### Principles
Use a primary key: Every table in TiDB should have a primary key defined. The primary key should be unique and should be used to identify individual rows in the table. By default, TiDB will use the primary key as the clustering index, which determines the physical ordering of rows in the table.

Choose the right secondary index: Secondary indexes can be used to improve the performance of queries that filter or sort data based on columns other than the primary key. When choosing which columns to index, consider the selectivity of the column (how many distinct values it has compared to the total number of rows in the table) and the frequency of queries that filter or sort based on that column.

Use covering indexes: As mentioned in the question, table access after an index scan can be expensive in a distributed database like TiDB. One way to avoid this is to use covering indexes, which include all the columns needed for a query in the index itself. This allows the query to be satisfied entirely from the index, without the need to access the underlying table.

Be careful with composite indexes: Composite indexes (indexes that include multiple columns) can be useful for queries that filter or sort based on multiple columns. However, they can also be less selective than single-column indexes, which can lead to slower query performance. Additionally, composite indexes can be larger and more expensive to maintain than single-column indexes.



### Composed Index

The order of composed index is critical.

Suppose we have a table named orders with the following columns:
```sql
order_id INT PRIMARY KEY,
customer_id INT,
order_date DATE,
status VARCHAR(20),
total_amount DECIMAL(10,2)
```
And we want to create a composed index on (customer_id, order_date, status).

```sql
CREATE INDEX idx_customer_order_status ON orders(customer_id, order_date, status);
```

Now suppose we have the following query:
```sql
SELECT order_id, total_amount 
FROM orders 
WHERE customer_id = 12345 AND order_date = '2022-01-01' AND status = 'Shipped';
```

This query can leverage the composed index we created above because the leading columns in the index are used in the WHERE condition with equal conditions, and the last column in the index is used in a non-equal condition:
```sql
customer_id = 12345 AND order_date = '2022-01-01' AND status = 'Shipped'
```

The composed index can satisfy this query by first finding the entries with customer_id = 12345 and order_date = '2022-01-01', and then scanning the entries with status = 'Shipped'.

Here's the sample code for creating the composed index and running the above query in Python:
```python
import mysql.connector

# Connect to the TiDB server
cnx = mysql.connector.connect(user='user', password='password', host='host', database='database')

# Create the composed index
cursor = cnx.cursor()
cursor.execute('CREATE INDEX idx_customer_order_status ON orders(customer_id, order_date, status);')
cursor.close()

# Run the query with the composed index
cursor = cnx.cursor()
query = 'SELECT order_id, total_amount FROM orders WHERE customer_id = %s AND order_date = %s AND status = %s'
params = (12345, '2022-01-01', 'Shipped')
cursor.execute(query, params)
for (order_id, total_amount) in cursor:
    print(f'Order ID: {order_id}, Total Amount: {total_amount}')
cursor.close()

# Close the connection
cnx.close()
```


### Anti-Pattern
here's an anti-pattern to avoid when designing indexes:

#### Over-indexing
It's possible to create too many indexes, which can actually hurt query performance by slowing down write operations and using up valuable memory. Indexes consume disk space and must be updated every time a table is modified, so it's important to strike a balance between the number of indexes and their usefulness. Before creating an index, ask yourself whether it will actually be used for queries or whether it duplicates the functionality of an existing index.
For example, suppose you have a table orders with columns customer_id, order_date, and order_status, and you create separate indexes on each individual column:

```sql
CREATE INDEX idx_orders_customer_id ON orders (customer_id);
CREATE INDEX idx_orders_order_date ON orders (order_date);
CREATE INDEX idx_orders_order_status ON orders (order_status);
```
This can be problematic if queries typically involve all three columns, as each index must be updated and searched separately. Instead, a single composed index covering all three columns would likely be more efficient.

Here's a modified example of the previous Python code that demonstrates this anti-pattern:

```python
import mysql.connector

# Establish a connection to the TiDB server
cnx = mysql.connector.connect(user='your_username', password='your_password',
                              host='your_hostname', database='your_database')

# Define individual indexes on the orders table for each column
cursor = cnx.cursor()
cursor.execute("CREATE INDEX idx_orders_customer_id ON orders (customer_id)")
cursor.execute("CREATE INDEX idx_orders_order_date ON orders (order_date)")
cursor.execute("CREATE INDEX idx_orders_order_status ON orders (order_status)")

# Query orders for a specific customer within a date range and with a particular status
cursor.execute("SELECT * FROM orders WHERE customer_id = %s AND order_date BETWEEN %s AND %s AND order_status = %s", (123, '2022-01-01', '2022-01-31', 'shipped'))
orders = cursor.fetchall()

# Close the cursor and the connection to the TiDB server
cursor.close()
cnx.close()
In this modified code, we've created separate indexes on each individual column of the orders table, which can lead to over-indexing if queries typically involve all three columns. This can hurt query performance by slowing down write operations and consuming memory unnecessarily.
```