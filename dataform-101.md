# Dataform 101

Dataform is an approachable, open source, multi-runner, declarative model to define, test and execute data pipelines.

It compiles sql queries and javascript configuration into a dependency graph and orchestrates execution as runner-compatible pipelines.

## Why dataform?

* accessible skill and tooling requirements (sql queries, javascript configuration)
* automated build and orchestration of pipelines
* minimal online dependency - connection to a local emulator or a cloud provider is required to execute unit tests on the query engine
* performant and extensible (Dataform is written in javascript and very fast)
* test harnesses for query logic and error-detection (ie. unit tests and sanity checks)
* developer interface via command line, VSCode extension, javascript API, REST API or Dataform website

Declarative SQL syntax to create a table from a select statement on other tables have existed for many years. For example:

``` sql
DROP TABLE IF EXISTS my_schema.my_table;

SELECT column_a, column_b
INTO my_schema.my_table
FROM table_a

-- or 

CREATE TABLE my_schema.my_table AS
SELECT column_a, column_b
FROM table_a
```

The user complexity has been around adding boilerplate to minimise downtime (avoiding the drop table) and managing the sequencing of the statements to ensure consistency and prevent locks. This boilerplate varies by warehouse technology and can become a distraction from the business objectives of the ETL process.

Some developers resorted to imperative logic to explicitly manage the dependencies between derived layers of data. For example, developers would use an orchestrator tool to define a pipeline that ensured that steps A, B and C ran in parallel but step D only ran after all of those steps completed.

Dataform enables high developer focus on the declarative paradigm by automatic inference the dependency graph from the declarative code.

## Getting Started

The Dataform CLI can be installed using NPM:

``` bash
npm i -g @dataform/cli
# confirm installation
dataform --version
```

Dataform provides an official vscode extension with automatic compilation, syntax highlighting and intellisense. Install the extension with this snippet.

``` bash
code --install-extension dataform.dataform
```

Dataform enables software engineering best practice with each developer refining the codebase on their laptop with no dependency on  infrastructure for the development environment. Since the warehouse products have different features available, Dataform requires your choice of warehouse technologies to be specified to ensure only supported features are used in your project. Dataform is compatible with the following warehouse technologies:

* bigquery
* postgresql
* redshift
* snowflake
* sqldatawarehouse

Choose a warehouse provider from the above list (for example "bigquery") and a project name (for example "my-project"). Default database is a required option for bigquery. Run this code to set variables.

``` bash
export WAREHOUSE=bigquery
export PROJECT_NAME=my-project
if [ "$WAREHOUSE" = "bigquery" ]; then
    export DEFAULT_DATABASE="--default-database ${PROJECT_NAME}"
fi
```

Create a new project using the command "dataform init" and create a hello world table.

``` bash
# dataform init <warehouse> [project-dir] [--default-database <if-bigquery>
dataform init $WAREHOUSE $PROJECT_NAME $DEFAULT_DATABASE
# create a hello_world table object
cd $PROJECT_NAME
echo 'js { type: "view" } select "hello world" as col_a' >> ./definitions/hello_world.sqlx
dataform compile --json
```

You now have a functioning project. When we deploy this project, Dataform will generate a warehouse-compatible action, such as:

``` sql
 CREATE OR REPLACE VIEW hello_world 
 AS 
 select "hello world" as col_a
```

Explicit actions can be created as the type "operations" [nb: plural] and are intended for DDL statements to create objects such as routines, user-defined functions or machine learning models.

``` dataform 
// definitions/hello_world.sqlx
config {
  type: "operations"
}
  CREATE OR REPLACE VIEW dataform.hello_world 
  AS 
  select "hello world" as col_a
```

## Software engineering best practices

## Unit Test & formatting

Let's add a little complexity to the tables so that we have something to test.

Dataform is designed to enable software engineering best practices with each developer in an isolated developer sandbox. Dataform test has lightweight dependencies on the warehouse platform, you will need a connect to your sandbox to use the "test" command. To connect your IDE to the sandbox, you would ideally use a local emulator but likely will need a cloud resource, service account and generate a Dataform credentials file. Run the init-creds command and Dataform will guide you to create a credentials file.

``` bash
dataform init-creds $WAREHOUSE
```

Rename your Dataform credentials file and copy it into the project root directory. By default, Dataform expects a file named ".df-credentials.json" and the .gitignore file created by default includes this filename to prevent exposing the unencrypted credentials in source control.

If you use VSCode, add formatting as a build task then run the task "dataform format with the keyboard shortcut (⇧⌘B or crtl B)

``` bash
echo '{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "dataform format",
            "type": "shell",
            "command": "dataform format",
            "presentation": {
                "reveal": "silent",
                "panel": "dedicated"
            },
            "problemMatcher": [],
            "group": {
                "kind": "build",
                "isDefault": true
            }
        }
    ]
}' > ./.vscode/tasks.json
```

The following code generates sqlx files with the conventional config block and sql code complex enough for us to create a test. It also applies conventional formatting to the code for readability and enables continuous compile mode. In most circumstances, the dataform compile command generates errors when semi-colons exist at the end of queries [ :( ]).

``` bash
rm ./definitions/hello_world.sqlx
echo 'config { type: "view" } select "HeLlO WoRlD" as col_a' > ./definitions/view_a.sqlx
echo 'config { type: "view" } select lower(col_a) as col_b from view_a' > ./definitions/view_b.sqlx
```

The above code successfully compiles but may generate deployment errors resolving the fully qualified name of the view. By default, table objects inherit a database and schema from dataform.json and the table name is inherited from the name of the sqlx file. The config block can optionally include values to override the project configuration.

``` dataform d
// df_table.sqlx
config {
  type: "view"
  schema: "my_schema",
  database: "my_database"
}

SELECT 1 as my_column
```

Dataform provides the ref() function for defining dependencies between table objects. This function ensure the correct resolution of the fully qualified table name, complete and accurate ordering of dependencies between table objects, and enables optimisation strategies. The ref() function provides for a compulsory table argument and additional optional arguments for database and schema.

``` sql
# Dataform syntax with ref function
select * 
from ${ref( 
      { database: "my_database",
        schema: "my_schema",
        name: "my_table" }
    )
```

So the correct code will be:

``` bash
echo 'config { type: "view" } select lower(col_a) as col_b from ${ref("view_a")}' > ./definitions/view_b.sqlx
dataform format # formats the javascript and sql code
dataform compile
```

Now let's add a unit test:

``` bash
echo 'config { type: "test", dataset: "view_b" } input "view_a" {select "" as col_a} select "hello world" as col_b' > ./definitions/test_view_b.sqlx
echo 'inital test will fail'
dataform format
dataform test
echo 'config { type: "test", dataset: "view_b" } input "view_a" {select "HeLlO WoRlD" as col_a} select "hello world" as col_b' > ./definitions/test_view_b.sqlx
dataform compile
dataform format
echo 'now the dataform test will succeed'
dataform test
```

## Assertions: sanity checks

Views can be created over the table/view objects to identify rows where errors exist (if any). When dataform is run and tables are refreshed, any rows in these tables will be highlighted. Dataform names these views "assertions". By default, assertion views are created in the schema "dataform_assertions". A custom name is configurable in dataform.json or in the config block of the sqlx files. The sql files can optionally include assertions and Dataform will automatically generate an assertion view for each assertion in the config block. For example, this code will generate ~~three~~ two assertion views.

``` dataform 
// df_table.sqlx
config {
  type: "view",
  assertions: {
    // validate that data contains no values in these columns that are null
    nonNull: ["column_a", "column_b"], 
    // validate that data contains no duplicate combination of specific columns
    uniqueKey: ["column_a", "column_b"], 
    rowConditions: [
        "column_a IN (1, 2, 3)",
        "column_b > 0",
    ]
  }
}

SELECT
    1 as column_a,
    1 as column_b
```

This generates the assertion views:

* dataform_assertions.dataform_df_table_assertions_uniqueKey_0
* dataform_assertions.dataform_df_table_assertions_rowConditions (also includes the nonNull assertions)

Row condition assertions can be extended by javascript functions. For example, the rowConditions above could be refactored.

``` js 
// includes/positiveNumber.js
function positiveNumber(colName) {
  return `${colName} > 0`;
}

module.exports = {
  positiveNumber
};
```

``` dataform 
// definitions/df_table.sqlx
config {
  type: "view",
  schema: "my_schema",
  database: "my_database",
  assertions: {
    nonNull: ["column_a", "column_b"],
    uniqueKey: ["column_a", "column_b"],
    rowConditions: [
      positiveNumber.positiveNumber("column_a"),
      "column_a IN (1, 2, 3)"
    ]
  }
}

SELECT 
    1 as column_a,
    1 as column_b
```

Alternatively, an assertion view can be created with a discrete sqlx file. This enables the creation of an assertion in a specified, non-default schema and can be used to decompose rowCondition assertion views to provide clarity.

``` dataform definitions/df_table_assertions_rowConditions_status_positiveNumbers.sqlx
config {
  type: "assertion",
  schema: "my_assertion_schema"
}

SELECT 
FROM ${ref(df_table)}
WHERE NOT columnn_A > 0
```

## Declarations: flexible interfaces to independent systems

There are more types available in Dataform. Declarations are used to enable references to table objects using only a config block.

``` dataform 
// definitions/london_bicycles_cycle_hire.sqlx
config {
    type: "declaration",
    database: "bigquery-public-data",
    schema: "london_bicycles",
    name: "cycle_hire"
}
```

``` dataform 
// definitions/sources/london_bicycles_cycle_stations.sqlx
config {
    type: "declaration",
    database: "bigquery-public-data",
    schema: "london_bicycles",
    name: "cycle_stations"
}
```

Since declarations include no sqlx code, they can commonly be expressed more concisely and robustly in javascript files. For example, tables of data copied from independent source systems can be concisely described in one js file per data source.

``` dataform 
// definitions/sources/london_bicycles.js
declare({
    database: "bigquery-public-data",
    schema: "london_bicycles",
    names: [
      "cycle_hire",
      "cycle_stations"
    ]
});
```

Declarations can be referred to sqlx files using the ref() function. Derived tables with dependencies on declarations are compiled but declarations are not compiled.

``` dataform 
// definitions/bicyles_analysis.sqlx
config {
    type: "view"
}
select 
    start_station_name,
    max(duration) 
FROM ${ref("cycle_hire")} 
group by start_station_name
order by max(duration) desc 
limit 10
```

## Documentation

Config blocks can optionally include descriptions for tables, views and columns.

``` dataform 
// definitions/my_table.sqlx
config {
    type: "table",
    description: "This table contains summary stats by date aggregated by country",
    columns: {
        order_date: "Date of the order",
        order_id: "ID of the order",
        customer_id: "ID of the customer in the CRM",
        amount: "Money value of the order"
    }
}

select ...
```

Dataform deploys these descriptions as metadata on the deployed tables and views. Data warehouse platforms typically expose this metadata in information_schema views and catalog tools can be used to scrape metadata when required. If you deploy from the Dataform web app, the metadata is also stored in the integrated Dataform catalog.

## Tags

Tags can optionally be included in the config block.

``` dataform 
// definitions/my_table.sqlx
config{
    type: "table",
    tags: [ "daily" ] 
}
```

Tags can be used to filter the table/view objects refreshed with the command "run". Common tags include "hourly" and "daily".

``` bash
# run all the tables refreshe at daily intervals
dataform run --tags "daily"
# run all the tables refreshed at daily intervals
dataform run --tags "hourly"
```

## Optimisation strategies: run cache, append only, micro-batch and snapshots

Some warehouse systems include optimisation features relevant to the problem of creating or replacing tables. Dataform provides configuration options for optimisation strategies including useRunCache and incremental loads.

The useRunCache option is a boolean value defined in the dataform.json file. It is scoped to a project, is currently only applicable to bigquery and snowflake projects and requires a @dataform/core version of at least 1.6.11. Each sql statement generated by Dataform has inputs from the sql and javascript of the table and its precedents. If a given table/view exists in the warehouse from a previous run and the inputs have not changed since the previous run, then the outcome of recreating the table/view would be identical to current state. When this option is enabled, if Dataform can eliminate refreshing these tables/views from the run.

``` js
// dataform.json
{
    "warehouse": "bigquery",
    "useRunCache": true
}
```

A table can be configured to apply a where clause by changing the type from "table" to "incremental" and addition of a javascript template.

``` dataform
config {
  type: "incremental"
}

select ...

${ when(incremental(), `WHERE timestamp > (SELECT MAX(timestamp) FROM ${self()})`) }
```

### Software engineering tip: credential encryption

If you do intend to check in the credentials, encrypt the credential using gpg. Generate a secure password and use it with the following code to generate an encrypted credentials file called ".df-credentials.json.gpg" that can be securely included in source control.

``` bash
brew install --cask gpg-suite
gpg --symmetric --cipher-algo AES256 .df-credentials.json
```
