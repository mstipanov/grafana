+++
title = "Using MySQL in Grafana"
description = "Guide for using MySQL in Grafana"
keywords = ["grafana", "mysql", "guide"]
type = "docs"
[menu.docs]
name = "MySQL"
parent = "datasources"
weight = 7
+++

# Using MySQL in Grafana

> Only available in Grafana v4.3+.
>
> Starting from Grafana v5.1 you can name the time column *time* in addition to earlier supported *time_sec*. Usage of *time_sec*  will eventually be deprecated.

Grafana ships with a built-in MySQL data source plugin that allow you to query any visualize
data from a MySQL compatible database.

## Adding the data source

1. Open the side menu by clicking the Grafana icon in the top header.
2. In the side menu under the `Dashboards` link you should find a link named `Data Sources`.
3. Click the `+ Add data source` button in the top header.
4. Select *MySQL* from the *Type* dropdown.

### Data source options

Name | Description
------------ | -------------
*Name* | The data source name. This is how you refer to the data source in panels & queries.
*Default* | Default data source means that it will be pre-selected for new panels.
*Host* | The IP address/hostname and optional port of your MySQL instance.
*Database* | Name of your MySQL database.
*User* | Database user's login/username
*Password* | Database user's password

### Database User Permissions (Important!)

The database user you specify when you add the data source should only be granted SELECT permissions on
the specified database & tables you want to query. Grafana does not validate that the query is safe. The query
could include any SQL statement. For example, statements like `USE otherdb;` and `DROP TABLE user;` would be
executed. To protect against this we **Highly** recommmend you create a specific mysql user with restricted permissions.

Example:

```sql
 CREATE USER 'grafanaReader' IDENTIFIED BY 'password';
 GRANT SELECT ON mydatabase.mytable TO 'grafanaReader';
```

You can use wildcards (`*`)  in place of database or table if you want to grant access to more databases and tables.

## Macros

To simplify syntax and to allow for dynamic parts, like date range filters, the query can contain macros.

Macro example | Description
------------ | -------------
*$__time(dateColumn)* | Will be replaced by an expression to convert to a UNIX timestamp and rename the column to `time_sec`. For example, *UNIX_TIMESTAMP(dateColumn) as time_sec*
*$__timeEpoch(dateColumn)* | Will be replaced by an expression to convert to a UNIX timestamp and rename the column to `time_sec`. For example, *UNIX_TIMESTAMP(dateColumn) as time_sec*
*$__timeFilter(dateColumn)* | Will be replaced by a time range filter using the specified column name. For example, *dateColumn BETWEEN '2017-04-21T05:01:17Z' AND '2017-04-21T05:06:17Z'*
*$__timeFrom()* | Will be replaced by the start of the currently active time selection. For example, *'2017-04-21T05:01:17Z'*
*$__timeTo()* | Will be replaced by the end of the currently active time selection. For example, *'2017-04-21T05:06:17Z'*
*$__timeGroup(dateColumn,'5m')* | Will be replaced by an expression usable in GROUP BY clause. For example, *cast(cast(UNIX_TIMESTAMP(dateColumn)/(300) as signed)*300 as signed),*
*$__timeGroup(dateColumn,'5m',0)* | Same as above but with a fill parameter so all null values will be converted to the fill value (all null values would be set to zero using this example).
*$__unixEpochFilter(dateColumn)* | Will be replaced by a time range filter using the specified column name with times represented as unix timestamp. For example, *dateColumn > 1494410783 AND dateColumn < 1494497183*
*$__unixEpochFrom()* | Will be replaced by the start of the currently active time selection as unix timestamp. For example, *1494410783*
*$__unixEpochTo()* | Will be replaced by the end of the currently active time selection as unix timestamp. For example, *1494497183*

We plan to add many more macros. If you have suggestions for what macros you would like to see, please [open an issue](https://github.com/grafana/grafana) in our GitHub repo.

The query editor has a link named `Generated SQL` that show up after a query as been executed, while in panel edit mode. Click on it and it will expand and show the raw interpolated SQL string that was executed.

## Table queries

If the `Format as` query option is set to `Table` then you can basically do any type of SQL query. The table panel will automatically show the results of whatever columns & rows your query returns.

Query editor with example query:

{{< docs-imagebox img="/img/docs/v45/mysql_table_query.png" >}}

The query:

```sql
SELECT
  title as 'Title',
  user.login as 'Created By' ,
  dashboard.created as 'Created On'
 FROM dashboard
INNER JOIN user on user.id = dashboard.created_by
WHERE $__timeFilter(dashboard.created)
```

You can control the name of the Table panel columns by using regular `as ` SQL column selection syntax.

The resulting table panel:

![](/img/docs/v43/mysql_table.png)

## Time series queries

If you set `Format as` to `Time series`, for use in Graph panel for example, then the query must return a column named `time` that returns either a sql datetime or any numeric datatype representing unix epoch.
Any column except `time` and `metric` is treated as a value column.
You may return a column named `metric` that is used as metric name for the value column.

**Example with `metric` column:**

```sql
SELECT
  $__timeGroup(time_date_time,'5m'),
  min(value_double),
  'min' as metric
FROM test_data
WHERE $__timeFilter(time_date_time)
GROUP BY time
ORDER BY time
```

**Example using the fill parameter in the $__timeGroup macro to convert null values to be zero instead:**

```sql
SELECT
  $__timeGroup(createdAt,'5m',0),
  sum(value_double) as value,
  measurement
FROM test_data
WHERE
  $__timeFilter(createdAt)
GROUP BY time, measurement
ORDER BY time
```

**Example with multiple columns:**

```sql
SELECT
  $__timeGroup(time_date_time,'5m'),
  min(value_double) as min_value,
  max(value_double) as max_value
FROM test_data
WHERE $__timeFilter(time_date_time)
GROUP BY time
ORDER BY time
```

Currently, there is no support for a dynamic group by time based on time range & panel width.
This is something we plan to add.

## Templating

This feature is currently available in the nightly builds and will be included in the 5.0.0 release.

Instead of hard-coding things like server, application and sensor name in you metric queries you can use variables in their place. Variables are shown as dropdown select boxes at the top of the dashboard. These dropdowns makes it easy to change the data being displayed in your dashboard.

Checkout the [Templating]({{< relref "reference/templating.md" >}}) documentation for an introduction to the templating feature and the different types of template variables.

### Query Variable

If you add a template variable of the type `Query`, you can write a MySQL query that can
return things like measurement names, key names or key values that are shown as a dropdown select box.

For example, you can have a variable that contains all values for the `hostname` column in a table if you specify a query like this in the templating variable *Query* setting.

```sql
SELECT hostname FROM my_host
```

A query can returns multiple columns and Grafana will automatically create a list from them. For example, the query below will return a list with values from `hostname` and `hostname2`.

```sql
SELECT my_host.hostname, my_other_host.hostname2 FROM my_host JOIN my_other_host ON my_host.city = my_other_host.city
```

To use time range dependent macros like `$__timeFilter(column)` in your query the refresh mode of the template variable needs to be set to *On Time Range Change*.

```sql
SELECT event_name FROM event_log WHERE $__timeFilter(time_column)
```

Another option is a query that can create a key/value variable. The query should return two columns that are named `__text` and `__value`. The `__text` column value should be unique (if it is not unique then the first value is used). The options in the dropdown will have a text and value that allows you to have a friendly name as text and an id as the value. An example query with `hostname` as the text and `id` as the value:

```sql
SELECT hostname AS __text, id AS __value FROM my_host
```

You can also create nested variables. For example if you had another variable named `region`. Then you could have
the hosts variable only show hosts from the current selected region with a query like this (if `region` is a multi-value variable then use the `IN` comparison operator rather than `=` to match against multiple values):

```sql
SELECT hostname FROM my_host  WHERE region IN($region)
```

### Using Variables in Queries

From Grafana 4.3.0 to 4.6.0, template variables are always quoted automatically so if it is a string value do not wrap them in quotes in where clauses.

From Grafana 4.7.0, template variable values are only quoted when the template variable is a `multi-value`.

If the variable is a multi-value variable then use the `IN` comparison operator rather than `=` to match against multiple values.

There are two syntaxes:

`$<varname>`  Example with a template variable named `hostname`:

```sql
SELECT
  UNIX_TIMESTAMP(atimestamp) as time,
  aint as value,
  avarchar as metric
FROM my_table
WHERE $__timeFilter(atimestamp) and hostname in($hostname)
ORDER BY atimestamp ASC
```

`[[varname]]`  Example with a template variable named `hostname`:

```sql
SELECT
  UNIX_TIMESTAMP(atimestamp) as time,
  aint as value,
  avarchar as metric
FROM my_table
WHERE $__timeFilter(atimestamp) and hostname in([[hostname]])
ORDER BY atimestamp ASC
```

#### Disabling Quoting for Multi-value Variables

Grafana automatically creates a quoted, comma-separated string for multi-value variables. For example: if `server01` and `server02` are selected then it will be formatted as: `'server01', 'server02'`. Do disable quoting, use the csv formatting option for variables:

`${servers:csv}`

Read more about variable formatting options in the [Variables]({{< relref "reference/templating.md#advanced-formatting-options" >}}) documentation.

## Annotations

[Annotations]({{< relref "reference/annotations.md" >}}) allow you to overlay rich event information on top of graphs. You add annotation queries via the Dashboard menu / Annotations view.

**Example query using time column with epoch values:**

```sql
SELECT
  epoch_time as time,
  metric1 as text,
  CONCAT(tag1, ',', tag2) as tags
FROM
  public.test_data
WHERE
  $__unixEpochFilter(epoch_time)
```

**Example query using time column of native sql date/time data type:**

```sql
SELECT
  native_date_time as time,
  metric1 as text,
  CONCAT(tag1, ',', tag2) as tags
FROM
  public.test_data
WHERE
  $__timeFilter(native_date_time)
```

Name | Description
------------ | -------------
time | The name of the date/time field. Could be a column with a native sql date/time data type or epoch value.
text | Event description field.
tags | Optional field name to use for event tags as a comma separated string.

## Alerting

Time series queries should work in alerting conditions. Table formatted queries is not yet supported in alert rule conditions.

## Configure the Datasource with Provisioning

It's now possible to configure datasources using config files with Grafana's provisioning system. You can read more about how it works and all the settings you can set for datasources on the [provisioning docs page](/administration/provisioning/#datasources)

Here are some provisioning examples for this datasource.

```yaml
apiVersion: 1

datasources:
  - name: MySQL
    type: mysql
    url: localhost:3306
    database: grafana
    user: grafana
    password: password
```
