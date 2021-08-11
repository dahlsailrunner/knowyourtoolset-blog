---
title: "Manual ORM via ADO.Net when other platforms aren't available" # Title of the blog post.
date: 2015-04-25T14:42:31-05:00 # Date of post creation.
description: "A way to achieve some ORM functionality without using Entity Framework or NHibernate" # Description used for search engine.
codeMaxLines: 10 # Override global value for how many lines within a code block before auto-collapsing.
codeLineNumbers: false # Override global value for showing of line numbers within code block.
figurePositionShow: true # Override global value for showing the figure label.
tags:
  - ADO.NET
  - C#
# comment: false # Disable comment if false.
---

{{% notice note Hindsight %}}
I wrote this article before I had heard of or come across the excellent Dapper library.  Nowadays I highly recommend that.  :)
{{% /notice %}}

Sometimes you come across situations where it is inappropriate to use a higher-level ORM utility such as EntityFramework, NHibernate or others. There are many very good situations in which these tools can be used, and this article is *not* an evaluation of them nor does it prescribe situations in which they should not be used. Rather, if you find yourself in a situation where this occurs you may still want to make your database access work with your strongly-typed classes using ADO.Net. With a little bit of “helper” code, you can make this a snap.

This post will show how to turn ADO.Net DataTables and DataSets into strongly typed custom .Net classes using a little bit of reflection. I will focus on the ADO.Net methods that return a result set of some sort or another. The starting point for my approach (and the initial idea) came from an article I read in CODE Magazine by Paul D Sheriff titled [Creating Collections of Entity Objects](http://www.codemag.com/Article/1305031).

{{% notice info "Regarding Performance" %}}
You may have come across information that states that reflection is slow and can cause performance problems. I work with a fairly large-scale enterprise application and we use code like this quite a bit without any real perceived pain. Unless you are dealing with very heavy loads running through code like this, you probably won’t even notice a difference between it and more traditionally hard-coded ADO.NET DataTable work. A general rule for me is to “program it easily first and evaluate performance when needed.” (see 
[Rule 8 of my Guiding Principles]({{< ref "/2015/03/guiding-principles-for-programmers#8-write-efficient-code--and-address-performance-when-you-need-to" >}})).
{{% /notice %}}

For this discussion, I will start with an example and then expand it to additional cases.

The example is that I want to execute a stored procedure and have its result set returned as a strongly type list to a calling method. For simplicity I’ll just be discussing from the standard AdventureWorks database.

Here is a custom stored proc that I’ll start with:
```sql
CREATE PROCEDURE [Person].[spGetSomeContacts]   
AS
SELECT TOP 100 
     c.ContactID    
    ,c.FirstName    
    ,c.LastName    
    ,c.EmailAddress
    ,c.Phone   
    ,c.ModifiedDate 
FROM Person.Contact c
```
This is admittedly a simple and not-likely-production proc, but it will serve our example just fine. So now (again, just as an example), let’s say we have a C# class that looks something like this:
```csharp
public class CustomContact 
{
    public int ContactId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string EmailAddress { get; set; }
    public DateTime ModifiedDate { get; set; }
    public bool IsBrandNew { get; set; }
}
```
Couple of key points to note here:

* The `Phone` field selected in the stored proc doesn’t seem to have a corresponding property on the class.
* The `IsBrandNew` property on the class doesn’t seem to have a corresponding field that will be coming from the database.
* All of the other properties match up by name (case-insensitive) between the stored proc and the class.

In order for us to leverage ADO.Net and get this data into our class, we will use its `SqlDataAdapter.Fill` method which gives us a `DataSet` with one or more `DataTables` in it.

The first step in our process involves creating a couple of simple overloads — one which can return a DataSet and one which can return a DataTable. See the overloads below.

```csharp
ublic void Execute(out DataSet resultSet, out int affectedRows)
{
    resultSet = new DataSet();
    affectedRows = 0;
 
    try
    {
        using (var sqlDataAdapter = new SqlDataAdapter(Command))
        {
            affectedRows = sqlDataAdapter.Fill(resultSet);                    
        }
    }
    catch (SqlException sqlErr)
    {
        throw new StoredProcException("SQL Exception occurred!!", Command.CommandText, InputsString, sqlErr);
    }
    catch (Exception ex) // some other kind of exception occurred!!
    {
        throw new StoredProcException("General Exception occurred!!", Command.CommandText, InputsString, ex);
    }
}
 
public void Execute(out DataTable resultSet)
{
    DataSet ds;
    int affectedRows;
 
    Execute(out ds, out affectedRows);
    resultSet = ds.Tables[0];
    ds.Dispose();            
}
```

Now that let’s write a method that can take a DataTable and return a list of a generic type — this is the meat of everything that will follow.

```csharp
public class CollectionHelper
{
    public static List<T> BuildCollection<T>(Type type, DataTable table)
    {
        var ret = new List<T>();
         
        var props = type.GetProperties();
 
        foreach (DataRow row in table.Rows)
        {                
            var entity = Activator.CreateInstance<T>();
 
            //Set all properties from the column names
            //NOTE: This assumes your column names are the same name as your class property names
            foreach (var col in props)
            {
                if (!table.Columns.Contains(col.Name))
                    continue;  // nothing to do with this column
 
                try
                {
                    if (!col.PropertyType.FullName.StartsWith("System"))
                    {
                        DeserializeColumnOutputIntoCustomType(row, col, entity);
                        continue;
                    }
 
                    if (row[col.Name].Equals(DBNull.Value))
                    {
                        col.SetValue(entity, null);
                        continue;
                    }
 
                    if (col.PropertyType == typeof (bool))
                    {                            
                        col.SetValue(entity, (row[col.Name].ToString() == "1" || row[col.Name].ToString() == bool.TrueString));
                        continue;
                    }
 
                    if (IsTypeNumeric(col.PropertyType))
                    {                            
                        col.SetValue(entity, string.IsNullOrEmpty(row[col.Name].ToString()) ? 0 : Convert.ChangeType(row[col.Name].ToString(), col.PropertyType), null);
                        continue;
                    }
 
                    if (col.PropertyType.Name.StartsWith("Nullable"))
                    {
                        var colType = Nullable.GetUnderlyingType(col.PropertyType);
                        if (IsTypeNumeric(colType))
                        {
                            col.SetValue(entity, Convert.ChangeType(row[col.Name].ToString(), colType), null);
                            continue;
                        }
 
                        if (colType == typeof (bool))
                        {
                            col.SetValue(entity, row[col.Name].ToString() == "1");
                            continue;
                        }
                         
                        //non-numeric, non-bool nullables
                        col.SetValue(entity, Convert.ChangeType(row[col.Name].ToString(), Nullable.GetUnderlyingType(col.PropertyType)), null);                            
                        continue;
                    }
                     
                    col.SetValue(entity, col.PropertyType.IsEnum ? Enum.ToObject(col.PropertyType, Convert.ToInt32(row[col.Name].ToString()))
                            : Convert.ChangeType(row[col.Name].ToString(), col.PropertyType), null);
 
                }
                catch (Exception ex)
                {
                    throw new Exception(string.Format("Failed building collection. Setting Property{0} to value {1}", col.Name, row[col.Name]), ex);
                }
            }
 
            ret.Add(entity);
        }
 
        return ret;
    }
 
    private static bool IsTypeNumeric(Type type)
    {
        return type == typeof(int) || type == typeof(short) || type == typeof(long) || type == typeof(double) || type == typeof(decimal) || type == typeof(float);
    }
 
    private static void DeserializeColumnOutputIntoCustomType<T>(DataRow row, PropertyInfo col, T entity)
    {
        if (string.IsNullOrEmpty(row[col.Name].ToString()))
            return;  // nothing to deserialize
 
        var outputType = col.PropertyType.UnderlyingSystemType;
        var handler = new XmlSerializer(outputType);
        using (var reader = new StringReader(row[col.Name].ToString()))
            col.SetValue(entity, handler.Deserialize(reader));
         
    }
}
```

It’s definitely worth explaining some of the above method a bit. The first things the method does is to create an empty list of type `T` and get 
the list of properties on type `T` (in the props variable). Then in loops through the `DataTable`’s rows and creates a new `T` to add to the list. At this 
point it loops through the properties on the object to see if the `DataTable` contains a column with the same name as the current property. If it doesn’t, 
it simply moves on to the next property. If it does, we set the value of the current `T`’s property to the value of the data column. Note that there is 
different logic for different C# types, as well as some specific handling for nullable types and null values. Of additional note is that if 
you have a complex type as one of your properties and that is returned by a stored procedure as xml text, this function will 
attempt to deserialize the object and set the property.

Now that we have a collection builder, now we can add an overload to our stored proc wrapper that will execute the proc and return us a strongly-typed generic list.

```csharp
public void Execute<T>(out List<T> outList)
{
    DataTable dt;
    Execute(out dt);
    outList = CollectionHelper.BuildCollection<T>(typeof (T), dt);
}
```

So now the application code that calls to get a list of CustomContacts (our initial example case) becomes very simple:

```csharp
List<CustomContact> contactList;
exsp.Execute(out contactList);
// "exsp" is actually an instance of the stored proc wrapper class that exposes the "Execute" overloads above.  
```

Note that we can use the exact same technique for any stored proc returning a list of objects now. Sweet!!!

To go further we can add a couple of additional overloads which may help us.
1. Get a single object
```csharp
public void Execute<T>(out T output, Tid myTid = null)
{
    List<T> tempList;
    Execute(out tempList);
    return tempList.FirstOrDefault();
}
```
2. Updating the properties on an already-instantiated object. This is a little trickier, but is relatively 
simple using the approach laid out above. Stay tuned for a more complete discussion of the entire stored proc wrapper — 
which will include the exception handling hinted at by the code above as well as handling of input parameters. 🙂