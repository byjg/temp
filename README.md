# Database Relational AnyDataset

[![Opensource ByJG](https://img.shields.io/badge/opensource-byjg.com-brightgreen.svg)](http://opensource.byjg.com)
[![Build Status](https://travis-ci.org/byjg/anydataset-db.svg?branch=master)](https://travis-ci.org/byjg/anydataset-db)

## Description

Anydataset Database Relational abstraction. Anydataset is an agnostic data source abstraction layer in PHP.

See more about Anydataset [here](https://github.com/byjg/anydataset).

## Features

- Connection based on URI
- Support and fix code tricks with several databases (MySQL, PostgresSql, MS SQL Server, etc)
- Natively supports Query Cache by implementing a PSR-6 interface
- Supports Connection Routes based on regular expression against the queries, that's mean a select in a table should be 
executed in a database and in another table should be executed in another (even if in different DB)

## Connection Based on URI

The connection string for databases is based on URL. 

See below the current implemented drivers:

| Database      | Connection String                                     | Factory
| ------------- | ----------------------------------------------------- | -------------------------  |
| Sqlite        | sqlite:///path/to/file                                | getDbRelationalInstance()  |
| MySql/MariaDb | mysql://username:password@hostname:port/database      | getDbRelationalInstance()  |
| Postgres      | psql://username:password@hostname:port/database       | getDbRelationalInstance()  |
| Sql Server    | dblib://username:password@hostname:port/database      | getDbRelationalInstance()  |
| Oracle (OCI)  | oci://username:password@hostname:port/database        | getDbRelationalInstance()  |
| Oracle (OCI8) | oci8://username:password@hostname:port/database       | getDbRelationalInstance()  |

```php
<?php
$conn = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance("mysql://root:password@10.0.1.10/myschema");
```

## Examples

### Basic Query

```php
<?php
$dbDriver = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance('mysql://username:password@host/database');
$iterator = $dbDriver->getIterator('select * from table where field = :param', ['param' => 'value']);
foreach ($iterator as $row) {
    // Do Something
    // $row->getField('field');
}
```

### Updating in Relational databases

```php
<?php
$dbDriver = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance('mysql://username:password@host/database');
$dbDriver->execute(
    'update table set other = :value where field = :param',
    [
        'value' => 'othervalue',
        'param' => 'value of param'
    ]
);
```

### Inserting and Get Id

```php
<?php
$dbDriver = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance('mysql://username:password@host/database');
$id = $dbDriver->executeAndGetId(
    'insert into table (field1, field2) values (:param1, :param2)',
    [
        'param1' => 'value1',
        'param2' => 'value2'
    ]
);
```

### Database Transaction

```php
<?php
$dbDriver = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance('mysql://username:password@host/database');
$dbDriver->beginTransaction();

// ... Do your queries

$dbDriver->commitTransaction(); // or rollbackTransaction()
```

### Cache results

You can easily cache your results with the DbCached class; You need to add to your project an
implementation of PSR-6. We suggested you add "byjg/cache".

```php
<?php
$dbDriver = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance('mysql://username:password@host/database');
$dbCached = new \ByJG\AnyDataset\Db\DbCached(
    $dbDriver,              // The connection string
    $psrCacheEngine,        // Any PSR-6 (Cache) implementation
    30                      // TTL (in seconds)
);

// Use the DbCached instance instead the DbDriver
$iterator = $dbCached->getIterator('select * from table where field = :param', ['param' => 'value']);
```

### Load balance and connection pooling 

The API have support for connection load balancing, connection pooling and persistent connection.

There is the Route class an DbDriverInterface implementation with route capabilities. Basically you have to define 
the routes and the system will choose the proper DbDriver based on your route definition.

Example:

```php
<?php
$dbDriver = new \ByJG\AnyDataset\Db\Route();

// Define the available connections (even different databases)
$dbDriver
    ->addDbDriverInterface('route1', 'sqlite:///tmp/a.db')
    ->addDbDriverInterface('route2', 'sqlite:///tmp/b.db')
    ->addDbDriverInterface('route3', 'sqlite:///tmp/c.db')
;

// Define the route
$dbDriver
    ->addRouteForWrite('route1')
    ->addRouteForRead('route2', 'mytable')
    ->addRouteForRead('route3')
;

// Query the database
$iterator = $dbDriver->getIterator('select * from mytable'); // Will select route2
$iterator = $dbDriver->getIterator('select * from othertable'); // Will select route3
$dbDriver->execute('insert into table (a) values (1)'); // Will select route1;
```  

The possible route types are:

- addRouteForWrite($routeName, $table = null): Filter any insert, update and delete. Optional specific table;
- addRouteForRead($routeName, $table = null): Filter any select. Optional specific table;
- addRouteForInsert($routeName, $table = null): Filter any insert. Optional specific table;
- addRouteForDelete($routeName, $table = null): Filter any delete. Optional specific table;
- addRouteForUpdate($routeName, $table = null): Filter any update. Optional specific table;
- addRouteForFilter($routeName, $field, $value): Filter any WHERE clause based on FIELD = VALUE
- addCustomRoute($routeName, $regEx): Filter by a custom regular expression. 

### Connecting To MySQL via SSL
    
(Read here https://gist.github.com/byjg/860065a828150caf29c20209ecbd5692 about create server mysql)

```php
<?php
$sslCa = "/path/to/ca";
$sslKey = "/path/to/Key";
$sslCert = "/path/to/cert";
$sslCaPath = "/path";
$sslCipher = "DHE-RSA-AES256-SHA:AES128-SHA";
$verifySsl = 'false';  // or 'true'. Since PHP 7.1.

$db = \ByJG\AnyDataset\Db\Factory::getDbRelationalInstance(
    "mysql://localhost/database?ca=$sslCa&key=$sslKey&cert=$sslCert&capath=$sslCaPath&verifyssl=$verifySsl&cipher=$sslCipher"
);

$iterator = $db->getIterator('select * from table where field = :value', ['value' => 10]);
foreach ($iterator as $row) {
    // Do Something
    // $row->getField('field');
}
```


### Using IteratorFilter in order to get the SQL

You can use the IteratorFilter object to make easier create SQL

```php
<?php
// Create the IteratorFilter instance
$filter = new \ByJG\AnyDataset\Core\IteratorFilter();
$filter->addRelation('field', \ByJG\AnyDataset\Enum\Relation::EQUAL, 10);

// Generate the SQL
$param = [];
$formatter = new \ByJG\AnyDataset\Db\IteratorFilterSqlFormatter();
$sql = $formatter->format(
    $filter->getRawFilters(),
    'mytable',
    $param,
    'field1, field2'
);

// Execute the Query
$iterator = $db->getIterator($sql, $param);
```

### Using IteratorFilter with Literal values

Sometimes you need an argument as a Literal value like a function or an explicit conversion. 

In this case you have to create a class that expose the "__toString()" method

```php
<?php

// The class with the "__toString()" exposed
class MyLiteral
{
    //...
    
    public function __toString() {
        return "cast('10' as integer)";
    }
}

// Create the literal instance
$literal = new MyLiteral();

// Using the IteratorFilter:
$filter = new \ByJG\AnyDataset\Core\IteratorFilter();
$filter->addRelation('field', \ByJG\AnyDataset\Core\Enum\Relation::EQUAL, $literal);
```


## Install

Just type: 

```bash
composer require "byjg/anydataset=4.0.*"
```

## Running Unit tests


```bash
vendor/bin/phpunit
```

## Running database tests

Run integration tests require you to have the databases up and running.

The easiest way to run the tests is:

**Prepare the environment**

```php
npm i
node_modules/.bin/usdocker --refresh
node_modules/.bin/usdocker -v --no-link mssql up
node_modules/.bin/usdocker -v --no-link mysql up
node_modules/.bin/usdocker -v --no-link postgres up
```

**Run the tests**


```
phpunit testsdb/PdoMySqlTest.php 
phpunit testsdb/PdoSqliteTest.php 
phpunit testsdb/PdoPostgresTest.php 
phpunit testsdb/PdoDblibTest.php 
```

Optionally you can set the password for Mysql and PostgresSQL

```bash
export MYSQL_PASSWORD=newpassword    # use '.' if want have a null password
export PSQL_PASSWORD=newpassword     # use '.' if want have a null password
```

----
[Open source ByJG](http://opensource.byjg.com)
