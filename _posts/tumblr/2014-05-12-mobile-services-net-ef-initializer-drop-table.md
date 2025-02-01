---
title: EF development-time initializer for your .NET-based mobile service
date: '2014-05-12T18:06:00-07:00'
tags:
- Azure Mobile Services
redirect_from:
- /post/85578106830/
- /post/85578106830/mobile-services-net-ef-initializer-drop-table/
---
<p><strong><em>Update: look at that&hellip; based on your feedback we&rsquo;ve baked this right into the product (starting with version 1.0.342 of our NuGet package). Check out <a href="http://azure.microsoft.com/en-us/documentation/articles/mobile-services-dotnet-backend-how-to-use-code-first-migrations/">this tutorial</a> for instructions on how to use the new initializers, and please disregard the code below. What&rsquo;s in the box works better!</em></strong></p>
<p>A few folks have encountered an issue while developing a .NET-based mobile service using our VS template or portal quickstart. If you publish your mobile service to the cloud and then make some changes to the model and re-publish, the EF initializer will fail complaining that it doesn&rsquo;t have sufficient permissions. The error might look like so in the Logs tab of the portal:</p>
~~~
Exception=System.InvalidOperationException: This operation requires a connection to the 'master' database. Unable to create a connection to the 'master' database because the original database connection has been opened and credentials have been removed from the connection string. Supply an unopened connection. ---> System.Data.SqlClient.SqlException: Login failed for user 'zXCBHhDWhTLogin'.
~~~
<p>This is the case because by default we use the <a href="http://msdn.microsoft.com/en-us/library/gg679604(v=vs.113).aspx">DropCreateDatabaseIfModelChanges</a> initializer, but the user under which your mobile service accesses your Azure SQL database doesn&rsquo;t have the permission to drop the database.</p>
<p>To work around that we have developed an alternative initializer that does largely the same thing, but works using the permission set we already have. <strong>Please exercise caution: this initializer will delete your data. It is intended to be used during development while you are experimenting with your model and database schema.</strong></p>
<p>Here is the example: instead of dropping the whole database, it just drops the tables used by your mobile service:</p>
~~~ csharp
public class DropCreateSchemaTablesIfModelChanges<TContext> : CreateDatabaseIfNotExists<TContext> where TContext : DbContext
{
    private string schemaName;

    public DropCreateSchemaTablesIfModelChanges(string schemaName)
    {
        this.schemaName = schemaName;
    }

    public override void InitializeDatabase(TContext context)
    {
        if (context == null)
        {
            throw new ArgumentException("Context is null");
        }

        if (String.IsNullOrEmpty(schemaName))
        {
            throw new ArgumentException("Please provide a non-empty schema name");
        }

        if (context.Database.Exists())
        {
            try
            {
                if (context.Database.CompatibleWithModel(true))
                {
                    return;
                }
            }
            catch
            {

            }
            finally
            {
                var command = @"
                    DECLARE @tableName NVARCHAR(500)
                    DECLARE cur cursor
                        FOR
                            SELECT '[' + TABLE_SCHEMA + '].[' + TABLE_NAME + ']'
                            FROM INFORMATION_SCHEMA.TABLES
                            WHERE TABLE_SCHEMA = @ms_schema
                    OPEN cur
                    FETCH NEXT FROM cur INTO @tableName
                        WHILE @@FETCH_STATUS = 0
                        BEGIN
                            EXEC('DROP TABLE ' + @tableName);
                            FETCH NEXT FROM cur INTO @tableName;
                        END
                    CLOSE cur
                    DEALLOCATE cur";

                var cs = context.Database.Connection.ConnectionString;

                using (var objConn = new SqlConnection(cs))
                {
                    objConn.Open();
                    var objCmd = new SqlCommand(command, objConn);
                    objCmd.Parameters.Add("@ms_schema", SqlDbType.NVarChar, 64);
                    objCmd.Parameters["@ms_schema"].Value = schemaName;
                    objCmd.ExecuteNonQuery();
                }
            }


        }

        // Make sure all the tables are there
        base.InitializeDatabase(context);

        this.Seed(context);
        context.SaveChanges();
    }

    protected virtual void Seed(TContext context)
    {
    }
}
~~~
<p>To use this initializer, create a class that inherits from it so you can specify a <strong>Seed</strong> method with your sample data:</p>
~~~ csharp
public class myAwesomeServiceInitializer : DropCreateSchemaTablesIfModelChanges<myAwesomeServiceContext>
{
    public myAwesomeServiceInitializer(string schemaName) : base(schemaName)
    {

    }

    protected override void Seed(myAwesomeServiceContext context)
    {
        List<TodoItem> todoItems = new List<TodoItem>
        {
            new TodoItem { Id = "1", Text = "First item", Complete = false },
            new TodoItem { Id = "2", Text = "Second item", Complete = false },
        };

        foreach (TodoItem todoItem in todoItems)
        {
            context.Set<TodoItem>().Add(todoItem);
        }

        base.Seed(context);
    }
}
~~~
<p>Then, put the following in your <strong>WebApiConfig.Register</strong> method:</p>
~~~ csharp
Database.SetInitializer(new myAwesomeServiceInitializer(ServiceSettingsDictionary.GetSchemaName()));
~~~
<p>Hope this helps!</p>
