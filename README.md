# Lab 03

In this laboratory we will see:

- How to insert data from a file to more than one table.
- how to extend entities using abstract entities
- how to extend a schema to create a new one, overwriting the mapping
- how to work with two schemas and databases that share the same model
- how to use imported data from one database to import it into another

## Pre requirements

### Create database for test

Create file "docker-compose.yaml"

```yaml
version: '3'
services:
  test:
    container_name: lambdaorm-test
    image: mysql:5.7
    restart: always
    environment:
      - MYSQL_DATABASE=test
      - MYSQL_USER=test
      - MYSQL_PASSWORD=test
      - MYSQL_ROOT_PASSWORD=root
    ports:
      - 3306:3306
```

Create MySql database for test:

```sh
docker-compose up -d
```

Create user and set character:

```sh
docker exec lambdaorm-test  mysql --host 127.0.0.1 --port 3306 -uroot -proot -e "CREATE DATABASE test2;"
docker exec lambdaorm-test  mysql --host 127.0.0.1 --port 3306 -uroot -proot -e "ALTER DATABASE test CHARACTER SET utf8 COLLATE utf8_general_ci;"
docker exec lambdaorm-test  mysql --host 127.0.0.1 --port 3306 -uroot -proot -e "ALTER DATABASE test2 CHARACTER SET utf8 COLLATE utf8_general_ci;"
docker exec lambdaorm-test  mysql --host 127.0.0.1 --port 3306 -uroot -proot -e "GRANT ALL ON *.* TO 'test'@'%' with grant option; FLUSH PRIVILEGES;"
```

### Install lambda ORM CLI

Install the package globally to use the CLI commands to help you create and maintain projects

```sh
npm install lambdaorm-cli -g
```

## Test

### Create project

will create the project folder with the basic structure.

```sh
lambdaorm init -w lab_03
```

position inside the project folder.

```sh
cd lab_03
```

### Complete Schema

In the creation of the project the schema was created but without any entity.
Add the Country entity as seen in the following example

```yaml
app:
  src: src
  data: data
  models: models
  defaultDatabase: mydb
databases:
  - name: mydb
    dialect: mysql
    schema: countries
    connection:
      host: localhost
      port: 3306
      user: test
      password: test
      database: test
      multipleStatements: true
      waitForConnections: true
      connectionLimit: 10
      queueLimit: 0
  - name: mydb2
    dialect: mysql
    schema: countries2
    connection:
      host: localhost
      port: 3306
      user: test
      password: test
      database: test2
      multipleStatements: true
      waitForConnections: true
      connectionLimit: 10
      queueLimit: 0
schemas:
  - name: countries
    entities:
      - name: Positions
        abstract: true
        properties:
          - name: latitude
            length: 16
          - name: longitude
            length: 16
      - name: Countries
        extends: Positions
        primaryKey: ["iso3"]
        uniqueKey: ["name"]
        properties:
          - name: name
            nullable: false
          - name: iso3
            length: 3
            nullable: false
          - name: iso2
            nullable: false
            length: 2
          - name: capital
          - name: currency
          - name: region
          - name: subregion
        relations:
          - name: states
            type: manyToOne
            composite: true
            from: iso3
            entity: States
            to: countryCode
      - name: States
        extends: Positions
        primaryKey: ["id"]
        uniqueKey: ["countryCode", "name"]
        properties:
          - name: id
            type: integer
            nullable: false
          - name: name
            nullable: false
          - name: countryCode
            nullable: false
            length: 3
  - name: countries2
    extends: countries
    excludeModel: true
    entities:
      - name: Positions
        abstract: true
        properties:
          - name: latitude
            mapping: LATITUDE
          - name: longitude
            mapping: LONGITUDE
      - name: Countries
        mapping: TBL_COUNTRIES
        properties:
          - name: name
            mapping: NAME
          - name: iso3
            mapping: ISO_3
          - name: iso2
            mapping: ISO_2
          - name: capital
            mapping: CAPITAL
          - name: currency
            mapping: CURRENCY
          - name: region
            mapping: REGION
          - name: subregion
            mapping: SUB_REGION
      - name: States
        mapping: TBL_STATES
        properties:
          - name: id
            mapping: ID
          - name: name
            mapping: NAME
          - name: countryCode
            mapping: COUNTRY_CODE			
```

### Update

```sh
lambdaorm update
```

the file model.ts will be created inside src/models/countries.

### Sync

```sh
lambdaorm sync
lambdaorm sync -n mydb2
```

It will generate:

- the Counties and States tables in database test and a status file "mydb-state.json" in the "data" folder.
- the TBL_COUNTRIES and TBL_STATES tables in database test2 and a status file "mydb2-state.json" in the "data" folder.

### Popuplate Data

then we execute

```sh
lambdaorm run -e "Countries.bulkInsert().include(p => p.states)" -d ./data.json -n mydb
lambdaorm run -e "Countries.bulkInsert().include(p => p.states)" -d ./data.json -n mydb2
```

test:

```sh
lambdaorm run -e "Countries.page(1,10).include(p => p.states)"
lambdaorm run -e "Countries.page(1,10).include(p => p.states)" -n mydb2
```

### Delete data in Countries and states in mydb2

```sh
lambdaorm run -e "States.deleteAll()" -n mydb2
lambdaorm run -e "Countries.deleteAll()" -n mydb2
```

### Export data from mydb

```sh
lambdaorm export  -n mydb
```

### Import in mydb2 from data exported from mydb

```sh
lambdaorm import -s ./mydb-export.json -n mydb2
```

### Drop

remove all tables from the schema and delete the state file, mydb-state.json

```sh
lambdaorm drop -n mydb
lambdaorm drop -n mydb2
```

## End

### Remove database for test

Remove MySql database:

```sh
docker-compose down
```
