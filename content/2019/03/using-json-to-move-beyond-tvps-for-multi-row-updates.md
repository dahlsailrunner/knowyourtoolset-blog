---
title: "Using JSON to Move Beyond TVPs for Multi-Row Updates" # Title of the blog post.
date: 2019-03-20T10:12:34-05:00 # Date of post creation.
description: "Article description." # Description used for search engine.
# thumbnail: "/images/path/thumbnail.png" # Sets thumbnail image appearing inside card on homepage.
# shareImage: "/images/path/share.png" # Designate a separate image for social media sharing.
codeMaxLines: 25 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
tags:
  - .NET 
  - ADO.NET
  - C#
  - SQL
# comment: false # Disable comment if false.
---

Many .NET applications will use Table-Valued-Parameters (TVPs) to send in a **set** of rows to be updated to a single stored procedure. Using this approach requires a decent amount of legwork to achieve:

* Define a `TYPE` in SQL Server to represent the table valued parameter
* Assign permissions to the TYPE so that it can be used within the proc
* Write the proc to include the TVP and use it appropriately
* Format an ADO.NET `DataTable` class in C# code (and make sure to get the type conversions right)

This is a LOT of overhead (IMHO) to achieve a set of rows being affected in a single sproc call.

Fortunately, there are alternatives that can make this easier to achieve with fewer artifacts (namely no `TYPE` and no special permissions are required): XML and JSON. This post will focus on the JSON technique, which is available with SQL Server 2016 and later.

Consider a table with the following columns:

* `AccountFieldId`: unique key for the table
* `AccountId`: foreign key to the account table
* `FieldId`: foreign key to a field table
* `Value`: value for a given field and a given account

We would like to update a batch of rows for a single account, and multiple new values for fields.

Desired parameters for the proc:

* `AccountId`
* JSON for the FieldIds and Values

The JSON should probably look something like this:
```json
[
  { "FieldId": 123,
    "FieldValue": "set-to-this-value"
  },
  { "FieldId": 456,
    "FieldValue": "here is another value"
  }
]
```
## Sproc Syntax
The basic strategy with sprocs doing a batch of updates is getting the JSON into a temp table (or table variable) so that your update statement just joins to the temp table you’ve created from the JSON. Here is the syntax that can accept the parameters mentioned above and load the JSON into a table variable, and then merge the results into the actual `AccountField` table:

```sql
CREATE PROCEDURE [dbo].[AddUpdateAccountFieldFromJson] (
     @AccountId UNIQUEIDENTIFIER   
    ,@JsonInput NVARCHAR(MAX)
)   
 AS 
 BEGIN
    SET NOCOUNT ON; 

    DECLARE @Source TABLE (	
	[AccountID] [uniqueidentifier] NOT NULL,
	[FieldID] [bigint] NOT NULL,
	[Value] [nvarchar](max) NULL
    )

    INSERT INTO @Source ([AccountID],[FieldID],[Value])
    SELECT @AccountId,[FieldId],[Value]
    FROM OPENJSON(@JsonInput)
    WITH (		
	    [FieldId] BIGINT
	   ,[FieldValue] NVARCHAR(MAX)                 
    )

MERGE AccountField AS target  
    USING (SELECT [AccountID],[FieldID],[Value]
           FROM @Source) AS source
    ON ( target.[AccountId] = source.[AccountId] 
         AND target.[FieldId] = source.[FieldId] )  
    WHEN MATCHED AND (source.[Value] <> target.[Value])THEN   
        UPDATE SET Value = source.Value
    WHEN NOT MATCHED THEN  
        INSERT ( [AccountId], [FieldId], [Value] )  
        VALUES ( @AccountId, source.FieldId, source.Value ); 
END
```

## C# Code Syntax
The final step in getting batch updates to work is having some code that can CALL the sproc above.

You may be able to do this with an anonymous class, but here is a class definition that can surface the JSON we’re after:

```csharp
public class FieldData
{
    public long FieldId { get; set; }
    public string FieldValue { get; set; }
}
```
Given that class, here is some code that uses Newtonsoft.JSON and Dapper to call the proc above with the inputs that we want:
```csharp
var accountId = "1234-4567-8901";
var listOfFieldValues = new List<FieldData> { 
    new FieldData { FieldId = 123, FieldValue = "set-to-this-value" },
    new FieldData { FieldId = 456, FieldValue = "here is another value" }
};

var jsonInput = JsonConvert.SerializeObject(listOfFieldValues);
db.Execute("AddUpdateAccountFieldFromJson", new { accountId, jsonInput }, commandType: CommandType.StoredProcedure); 
```

Voila!  :)