# Sample code for using prepared statement with TiDB and the anti-patterns

## Prepared but forget to deallocate

```python
import mysql.connector

# Establish a connection to the MySQL server
cnx = mysql.connector.connect(user='your_username', password='your_password',
                              host='your_hostname', database='your_database')

# Create a cursor object to execute SQL queries
cursor = cnx.cursor(prepared=True)

# Define a prepared statement for selecting data from a table
query = "SELECT * FROM some_table WHERE some_column = %s"
data = ('some_value',)

# Execute the prepared statement
cursor.execute(query, data)

# Fetch the results and print them
for row in cursor.fetchall():
    print(row)

# Oops, forgot to deallocate the prepared statement!
# cursor.deallocate()

# Close the cursor and connection to the server
cursor.close()
cnx.close()

```

In this example, we establish a connection to a MySQL server and create a cursor object with the prepared option set to True, indicating that we will be using prepared statements. We define a prepared statement to select data from a table with a parameterized query using the %s placeholder. We then execute the prepared statement with the value 'some_value' for the parameter.

However, we forget to deallocate the prepared statement using the cursor.deallocate() method, which could result in a memory leak or other issues if the program were to run for an extended period of time or if a large number of prepared statements were created.

To avoid this issue, it's important to always deallocate prepared statements after they have been used, using the cursor.deallocate() method.

If this code snippt is in a loop, the situation will be worse and casue more serious result like OOM.

```python
import mysql.connector

# Establish a connection to the MySQL server
cnx = mysql.connector.connect(user='your_username', password='your_password',
                              host='your_hostname', database='your_database')

# Create a cursor object to execute SQL queries
cursor = cnx.cursor(prepared=True)

# Define a list of values to use in the prepared statement
values = ['value1', 'value2', 'value3']

# Loop through the values and execute a prepared statement for each one
for value in values:
    # Define a prepared statement for selecting data from a table
    query = "SELECT * FROM some_table WHERE some_column = %s"
    data = (value,)

    # Execute the prepared statement
    cursor.execute(query, data)

    # Fetch the results and print them
    for row in cursor.fetchall():
        print(row)

    # Oops, forgot to deallocate the prepared statement!
    # cursor.deallocate()

# Close the cursor and connection to the server
cursor.close()
cnx.close()
```

In this example, we establish a connection to a MySQL server and create a cursor object with the prepared option set to True, indicating that we will be using prepared statements. We define a list of values to use in the prepared statement, and then loop through each value and execute a new prepared statement for each one.

However, we forget to deallocate the prepared statement using the cursor.deallocate() method inside the loop, which could result in a memory leak or other issues if the loop were to run for an extended period of time or if a large number of prepared statements were created.

To avoid this issue, it's important to always deallocate prepared statements after they have been used, using the cursor.deallocate() method, even if the statement is being created inside a loop.

