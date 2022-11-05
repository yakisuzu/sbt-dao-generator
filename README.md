# sbt-dao-generator

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.github.sbt-dao-generator/sbt-dao-generator/badge.svg)](https://maven-badges.herokuapp.com/maven-central/io.github.sbt-dao-generator/sbt-dao-generator)

`sbt-dao-generator` is the code generator plug-in for O/R Mapper Free. Even if you use any O/R mapper framework, DAOs are automatically generated by this plug-in functions.

## How to use plugin

Add this to your `project/plugins.sbt` file:

```scala
addSbtPlugin("io.github.sbt-dao-generator" % "sbt-dao-generator" % "1.2.0")
```

## How to configuration

Add this to your build.sbt file:

```scala
// JDBC Driver Class Name (required)
generator / driverClassName := "org.h2.Driver"

// JDBC URL (required)
generator / jdbcUrl := "jdbc:h2:file:./target/test"

// JDBC User Name (required)
generator / jdbcUser := "sa"

// JDBC Password (required)
generator / jdbcPassword := ""

// The Function that convert The Column Type Name to Property Type Name (Optional)
generator / propertyTypeNameMapper := {
  case "INTEGER" => "Int"
  case "VARCHAR" => "String"
  case "BOOLEAN" => "Boolean"
  case "DATE" | "TIMESTAMP" => "java.util.Date"
  case "DECIMAL" => "BigDecimal"
}

// More flexible `PropertyTypeNameMapper` (Optional)
// NOTE: Currently this is ignored when the `propertyTypeNameMapper` is defined since it keeps compatibility.
// NOTE: We plan to rename this setting to `propertyTypeNameMapper` in the next major version.
generator / advancedPropertyTypeNameMapper := {
  case (_, TableDesc(tableName, _, _), ColumnDesc("id", _, _, _, _)) => s"${tableName}Id"
  case (_, _, ColumnDesc(_, _, _, _, Some(remarks))) => remarks
  case ("INTEGER", _, _) => "Int"
  case ("VARCHAR", _, _) => "String"
  case ("BOOLEAN", _, _) => "Boolean"
  case ("DATE" | "TIMESTAMP", _, _) => "java.util.Date"
  case ("DECIMAL", _, _) => "BigDecimal"
}

// Schema Name (Optional, Default is None)
generator / schemaName := None,

// The Function for filtering the table to be processed (Optional, default is the following)
generator / tableNameFilter := { tableName: String => tableName.toUpperCase != "SCHEMA_VERSION"}

// The Function for converting Table Name to Class Name (Optional, default is the following)
generator / classNameMapper := { tableName: String =>
    Seq(StringUtil.camelize(tableName))
}

// e.g.) If you want to specify multiple output files, you can configure it as follows.
/*
generator / classNameMapper := {
  case "DEPT" => Seq("Dept", "DeptSpec")
  case "EMP" => Seq("Emp", "EmpSpec")
}
*/

// The Function for converting Column Name to Property Name (Optional, default is the following)
generator / propertyNameMapper := { columnName: String =>
    StringUtil.decapitalize(StringUtil.camelize(columnName))
}

// The Function that decides which Template Name for Model Name (Optional, defaults below)
generator / templateNameMapper := { className: String => "template.ftl" },

// e.g.) If you want to specify different templates for the model and spec, you can configure it as follows.
/*
generator / templateNameMapper := {
  case className if className.endsWith("Spec") => "template_spec.ftl"
  case _ => "template.ftl"
}
*/

// The Directory where template files are placed (Optional, default is the following)
generator / templateDirectory := baseDirectory.value / "templates"

// The Directory where source code is output (Optional, default is the following)
generator / outputDirectoryMapper := { className: String => (Compile / sourceManaged).value }

// e.g.) You can change the output destination directory for each class name dynamically.
/*
generator / outputDirectoryMapper := { className: String =>
  className match {
    case s if s.endsWith("Spec") => (Test / sourceManaged).value
    case s => (Compile / sourceManaged).value
  }
}
*/
```

## How to configure a model template

The supported template syntax is FTL(FreeMarker Template Language).Please refer to the official FreeMarker document for details.

- [FreeMarker](https://freemarker.apache.org)

**`templates/temlate.ftl`**

```ftl
case class ${className}(
<#list allColumns as column>
<#if column.nullable>
${column.propertyName}: Option[${column.propertyTypeName}]<#if column_has_next>,</#if>
<#else>
${column.propertyName}: ${column.propertyTypeName}<#if column_has_next>,</#if>
</#if>
</#list>
) {

}
```

You can use the following template contexts.

**Top level objects**

| Variable name | Type | Description |
|:-----------|:---|:---------|
| `tableName` | String | Table Name (`USER_NAME`)|
| `className`  | String | Class Name　(`UserName`). The string converted `tableName` by `classNameMapper` |
| `decapitalizedClassName` | String | Decapitalized Class Name (`userName`) |
| `primaryKeys` | `java.util.List<Column>` | Primary Keys |
| `columns` | `java.util.List<Column>` | Columns without Primary Keys |
| `allColumns` | `java.util.List<Column>` | Columns with Primary Keys |

**Column objects**

| Variable name | Type | Description |
|:-----------|:---|:---------|
| `columnName` | `String` | Column Name (`FIRST_NAME`) |
| `columnTypeName` | `String` | Column Type Name (`VARCHAR`, `DATETIME`, ...) |
| `propertyName` | `String` | Property Name (`firstName`). The string converted `columnName` by `propertyNameMapper` |
| `propertyTypeName` | `String` | Property Type Name (`String`, `java.util.Date`,...). The string converted `columnTypeName` by `propertyTypeNameMapper` |
| `capitalizedPropertyName` | `String` | Capitalized Property Name (`FirstName`) |
| `nullable` | `Boolean` | NULL is true |

## Code generation

- When processing all tables

```sh
$ sbt generator/generateAll
<snip>
[info] tableName = DEPT, generate file = /Users/sbt-user/myproject/target/scala-2.13/src_managed/Dept.scala
[info] tableName = EMP, generate file = /Users/sbt-user/myproject/target/scala-2.13/src_managed/Emp.scala
[success] Total time: 0 s, completed 2015/06/24 18:17:20
```

- When processing multiple tables

```sh
$ sbt generator/generateMany DEPT EMP
<snip>
[info] tableNames = EMP, DEPT
[info] tableName = DEPT, generate file = /Users/sbt-user/myproject/target/scala-2.13/src_managed/Dept.scala
[info] tableName = EMP, generate file = /Users/sbt-user/myproject/target/scala-2.13/src_managed/Emp.scala
[success] Total time: 0 s, completed 2015/06/24 18:17:20
```

- When processing one table

```sh
$ sbt generator/generateOne DEPT
<snip>
[info] tableName = DEPT
[info] tableName = DEPT, generate file = /Users/sbt-user/myproject/target/scala-2.13/src_managed/Dept.scala
[success] Total time: 0 s, completed 2015/06/24 18:17:20
```

If you want to run `generator/generateAll` at `sbt compile`, add the following to build.sbt:

```scala
Compile / sourceGenerators += (generator / generateAll).value
```

## How to migration from v1.0.4 to v1.0.8

The following parameters have been changed. Please change your project accordingly.

- projectSettings

| Modify type | Old name | New name |
|:---------|:---------------|:-----------------------|
| Modified | typeNameMapper | propertyTypeNameMapper |

- Top level objects

| Modify type | Old name | New name |
|:---------|:-----|:----------|
| Modified | name | className |
| Added | - | tableName |
| Modified | primaryKeysWithColumns | allColumns |

- Column objects

| Modify type | Old name | New name |
|:---------|:-----------|:---------------|
| Modified | columnType | columnTypeName |
| Modified | propertyType | propertyTypeName |



