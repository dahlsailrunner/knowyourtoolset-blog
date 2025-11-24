---
title: "Using XML as an Input Parameter to Stored Procedures in Favor of table-valued-parameters" # Title of the blog post.
date: 2015-03-30T15:58:53-05:00 # Date of post creation.
summary: "Article description." # Description used for search engine.
codeMaxLines: 20 # Override global value for how many lines within a code block before auto-collapsing.
tags:
  - .NET 
  - ADO.NET
  - C#
  - SQL
# comment: false # Disable comment if false.
---

A common challenge with stored procedures is ***how to pass in multiple rows to a single stored proc call***  — especially when you
know all of the new / updated rows at a single time.  Doing so can save on round trips as well as (sometimes) making
transaction handling easier.

## Failed Attempt 1:  Just use flat parameters

If you have only a couple of columns and a fixed and low number of rows, you might just add parameters like “row1col1”, “row1col2”, etc. While this ***would*** work, it falls down pretty rapidly under almost any real-world scenario.

## Failed Attempt 2:  Use Table-Valued Parameters

Another approach that you often hear about — especially if you do research from the T-SQL side of the world, is Table-Valued
Parameters. There’s a [pretty good post here](http://www.mssqltips.com/sqlservertip/2112/table-value-parameters-in-sql-server-2008-and-net-c/)
that covers the mechanics of how to do it with C# and SQL — what I want to focus on is why I’m not a big fan of the approach. Here are the big reasons:

* There is a decent amount of SQL to do: **you create a `type`** for a table and you also have to **grant permissions for it**. You also have to code your stored procs with the special “readonly” qualifier for those parameters.
* Then in C#, you need to use ADO.NET `DataTable`s to pass in the value — so you have to manually add columns for each column in your TVP, then you have to add the rows for each row you want to pass in.
* Ultimately, what this implies is that you make changes to THREE places whenever any of the columns needs to change in what you’re passing in:  the SQL `TYPE` you have created, the stored proc to handle the column, and the C# code that formats the parameters.

## Just Right:  XML as an input parameter

So the approach that I’ve been taking lately is to use XML as an input parameter.  The Linq-To-XML features in C# (as well as standard XML
serialization) lend themselves to quickly turning a `List<T>` into an XML string.  Then the XML features inside SQL can turn your passed-in
XML into a SQL table (I use table variables) that can be easily referenced in your stored proc.  In this case, you only have TWO places to
change when you have column changes, and you don’t have any special permissions or stored proc `READONLY` syntax to remember.

In the deliberately simple example below, I’ll show how to create some XML containing a couple of rows and add it to the input
parameters of a stored procedure using C#, and then read the XML into a table variable in SQL.  The hope is that this simple
example illustrates the concept well enough for you to leverage in your real-world applications.

### The XML

The XML for this example will look like the below.

```xml
<Attributes>
   <Attr AttrAliasName="FavoriteColor" AttrValue="Green"/>
   <Attr AttrAliasName="Status" AttrValue="ReadyToRock"/>
</Attributes>
```
### The C #

Basically your whole goal here is to get some XML content created and established as a parameter to a stored procedure. I’ve got some
notes on this code below the snippet.

```csharp
var inputRows = new XElement("Attributes");
 
inputRows.Add(new XElement("Attr", new XElement("AttrAliasName", "FavoriteColor"),
                                 new XElement("AttrValue", "Green")));    
 
inputRows.Add(new XElement("Attr", new XElement("AttrAliasName", "Status"),
                                 new XElement("AttrValue", "ReadyToRock"))); 
 
using (var sqlConnection = new  SqlConnection("mydataconnectionstring"))
{
    var storedProc = new SqlCommand("spUpdateMultipleRows", sqlConnection);
    storedProc.CommandType = CommandType.StoredProcedure;
 
    storedProc.Parameters.Add("@CustId", 1234);
    storedProc.Parameters.Add("@BeginDt", new DateTime(2015, 3,27);
    storedProc.Parameters.Add("@AttrXml", inputRows.ToString());
 
    var rowsAffected = storedProc.ExecuteNonQuery();
}
```

Notes:

* I’m using LinqToXml in the first few lines to create some rows.  In a real world situation, this would probably be in some kind of a loop (or create the XML by serializing an object!).
* Note the ease of setting the XML data as an XML input parameter on line 16.  It’s as simple as `inputRows.ToString()`.
* The `rowsAffected` variable above reflects the number of rows affected by your stored proc call.  You could evaluate it if you choose to make sure that the right number of rows were updated.

### The SQL

Given the above XML, you can use SQL syntax as shown below to read that into a table variable.

```sql
CREATE  PROCEDURE [dbo].[spUpdateMultipleRows]
    @CustomerId INT
   ,@BeginDt DATETIME
   ,@AttrXml XML
AS
 
DECLARE @tmpAttrVal TABLE
    (
     AttrAliasNm VARCHAR(30)
    ,AttrValTxt VARCHAR(1000)
    )
 
INSERT  INTO @tmpAttrVal
        SELECT  T.Attribute.value('./AttrAliasName[1]', 'VARCHAR(30)') AS AttrAliasNm
               ,T.Attribute.value('./AttrValue[1]', 'VARCHAR(1000)') AS AttrValTxt
        FROM    @AttrXml.nodes('//Attributes/Attr') AS T (Attribute)
```

I’d be lying to you if I said the XML syntax in SQL is easy to follow and intuitive (at least for me). Here are a few notes about the above SQL snippet:

* Note that the input type for the `@AttrXml` parameter is `XML`, not any kind of text input.
* The `@AttrXml.nodes(‘//Attributes/Attr’)` is an `XPath` query for the list of rows (nodes) that will be aliased as `T.Attribute` by nature of the “T (Attribute)” syntax.
* To peel off the attributes for an individual row, you use `<rowalias`>`.value('./<attributename>[1]' <sqltype>)` as the syntax, where `rowalias` comes from the previous bullet, `attributename` is the xml attribute you’re looking for, and `sqltype` is the sql type for that value.

Once you get your XML properly established in your table variable, you’re pretty much off to the races, and you can update/insert/delete rows
from target tables with joins to your table variable in whatever situation makes sense.  I’m assuming in this statement that you are
familiar with CRUD statements using JOIN clauses.  Lots of resources on those topics.

**The bottom line:**  As in most non-trivial situations, the right approach for your situation depends on lots of factors that may not be
covered by this post.  But maybe this gives you some ideas on how to attack the challenge of
multiple rows into a single proc call in a new and refreshing way.  Enjoy!
