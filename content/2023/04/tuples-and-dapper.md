---
title: "Using Tuples with Dapper" # Title of the blog post.
date: 2023-04-16T05:51:55-05:00 # Date of post creation.
description: "Using tuples as the generic arguments for Query calls with Dapper can avoid the need from some simple classes." # Description used for search engine.
codeMaxLines: 30 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: true # Override global value for showing of line numbers within code block.
thumbnail: "/images/tuple-dapper.png" # Sets thumbnail image appearing inside card on homepage.
shareImage: "/images/tuple-dapper.png" # Designate a separate image for social media sharing.
toc: true
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - C#
  - ASP.NET  
  - Dapper
  - SQLite
---

## Summary

Using tuples with Dapper for simple query results can avoid the need
to create classes or records that might otherwise bloat your code-base.

Here's a sample repo: [https://github.com/dahlsailrunner/tuple-with-dapper](https://github.com/dahlsailrunner/tuple-with-dapper)

## Background

When you have an existing database or management practices that don't
involve Entity Framework (Core or otherwise),
[Dapper](https://github.com/DapperLib/Dapper) can be a great
way to perform data access without the lower-level details of ADO.NET,
and it can map your query results directly to strongly-typed C# objects
like classes and records as long as the column names of your query results
match up with the property names of those classes and records.

Here's a sample SQL statement for the "happy case" - which could be used
in a string variable called `sqlForCustomers`:

```sql
SELECT Id, 
      Name, 
      FavoriteColor, 
      CreateDate
FROM Customer
```

Then you might define (or already have) a C# class (or record) like the following:

```C#
public class CustomerInCode
{
  public int Id { get; set; }
  public string Name { get; set; }
  public string FavoriteColor { get; set; }
  public DateTime CreateDate { get; set; }
}
```

Then you could invoke Dapper's `Query<>` method like this:

```C#
var customers = dbConnection.Query<CustomerInCode>(sqlForCustomers);
```

And then `customers` would be an `IEnumerable<CustomerInCode>` and each
item would have all of its properties set with the results of the query.
Awesome!

## Classes for Everything?

The above code shows the `CustomerInCode` class as the generic argument for
the `Query<T>` method of Dapper.

There are some situations where this might be troublesome:

* The column names of the table / query don't match up with good C# property names
* The class you would create is ONLY meant to represent the results of this single query and would have no other re-usability
* Changing the query (or adding another one) to use column aliases is not desirable (for whatever reason)

You might be creating class *for no other reason* than to receive the results of a
given query. You might wonder where to put this class, and it might add "bloat"
to your codebase.

## An Alternative: Use Tuples!

For simple result sets (say 2-6 columns, tops) an alternative to creating a
class/record only for the results of the query is to use a tuple.

Here's what that code would look like (with different column names on the
`customer` table):

```C#
var customers = dbConnection.Query<(int Id, string Name, string FavoriteColor)>
        (@"SELECT id, 
              customer_name, 
              fav_color, 
              create_date
           FROM customer");
var firstCust = customers.First();
var n = firstCust.Name;  // works
```

In this example, `customers` is a tuple of the form `(int Id, string Name, string FavoriteColor)`.

There are a couple of noteworthy items here:

* The columnn names of the results **do not** need to match the tuple property names
* The ***order*** of the items of the tuple absolutely matters -- you need to define
    tuple properties for each column from the beginning to the last one you need
  * Notice that I don't have a column for `create_date` in the tuple
  * If I ommitted the `string Name` property, the `customer_name` value would go
    into the `string FavoriteColor` property
  * If I made it `(int Id, DateTime CreateDate)` then it would actually fail because
    it would not be able to convert the `customer_name` column value into a `DateTime`

Here's a simple repo that uses SQLite that you can run and experiment with
if you want: [https://github.com/dahlsailrunner/tuple-with-dapper](https://github.com/dahlsailrunner/tuple-with-dapper)

## In Closing

Using tuples with Dapper is definitely not a one-size fits all solution that
should be used everywhere, but selective usage in situations where a simple and
small class would be created only for the results of the query may be a great
opportunity for keeping your code simpler than it otherwise might have been.

Happy coding!
