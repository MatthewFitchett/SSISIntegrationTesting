# SSISIntegrationTesting

This project gives an example of how to go about writing an automated test around an SSIS package.  The database is often the place where a lot of mocking is used in application testing as it can be hard to write good repeatable tests around it.  The reason for this is that often there are only a few living instances of a database (dev, qa, prod) and everyone just crams their changes into.  Normally they are not kept in source control and so are hard to know what state they are in at any one time.  

By pulling your database into source control and being able to build it as part of a test solves a number of problems:
  *) You can write full stack integration tests that include your database (no mocks needed which are often brittle and are tightly coupled to the implementation)
  *) Your database can be fully built by any developer so each person can work independently.  Trying out their changes locally before they deploy them to an integration environment
  *) Reduces test time as a developer finds out straight away if their database changes do not work
  *) Saves wasting time fixing "ghost" issues, when using a shared database a lot of headaches are caused by many developers changing the database.  Meaning you are running code against a different version of the database to what the code expects.  This slows down development and often means you are chasing your tail looking for why the code does not work
  
  From the above we can see the advantages that being able to fully build and test your database gives you.  So how do we go about writing integration tests for SSIS packages?
  
  SSIS packages are of course often dependent on databases, their most common use case is for an ETL job to move vast quantities of data into a data warehouse database.  Obviously, we cannot use this large dataset for test purposes.  This project lays out a proof of concept of how you could test an etl job that moves data between two databases.
  
  The MyProject SSIS project includes `Package1.dtsx`.  This is a very simple package that takes data from a table called products in a source database and puts it into a table called products in a destination database.  Obviously in a real world example the job would be more complex but it gives us something to work with.  The key to making the SSIS package testable is by parameterizing the connection strings that are set for the source and destination databases in the connection manager.  In the `Package1.dtsx` the source and destination connection string are passed in by the `Source_ConnectionString` and `Dest_ConnectionString` parameters respecitively.  If none are passed in then the defaults are used by the package.
  
  With the SSIS package set up as described we are now in a position to write a full integration test.  Before we go into how that is done lets see what would consitute fully testing this SSIS package:
  1. create a source database with test data
  2. create an empty destination database
  3. run the SSIS package
  4. assert the source database data has been copied to the destination database
  5. clear away all created databases
     
    
 If we could do the steps above then we would have a full repeatable end to end test.  
 
 So lets describe what the integration test does that can be found in the `PackageScenarios.cs` file in the SSISTests project.  Each line of the test is given with the full source code and then I've added a description underneath of what it does.  What is neat about this code is that it is very self describing and you can see exactly what the code is doing without having to read the description.  This is one of the key points to writing good code (make it readable!!).
 
 ```
 "Given a source database with one row in the products table"
                ._(() =>
                {
                    sourceDatabase = _testServer.CreateNew();
                    sourceDatabase.ExecuteScript("database.sql");
                    var connection = Database.OpenConnection(sourceDatabase.ConnectionString);
                    connection.Products.Insert(ProductCode: 1, ShippingWeight: 2f, ShippingLength: 3f,
                        ShippingWidth: 4f, ShippingHeight: 5f, UnitCost: 6f, PerOrder: 2);
                });
 ```
 
 The first step of the test creates a brand new database on the local db instance (which is automatically running if you install sql server) this will be used as our source databsae.  It then runs in our base schema which we have named `database.sql` (if you open the database.sql file you will see that this just consists of our products table).  We then use [Simple.Data](http://simplefx.org/simpledata/docs/) to insert a single product into our source databse.  We now have a source database set up and ready to go with a single product in the products table.
 
 ```
 "And an empty destination database with a products table"
                ._(() =>
                {
                    destDatabase = _testServer.CreateNew();
                    destDatabase.ExecuteScript("database.sql");
                });
 ```
 This step creates a brand-new database on the local db instance and runs in the `database.sql` schema (which is our products table).  This will be used as our destination database.
 
 
 ```
 "When I execute the migration package against the source and dest databases"
                ._(() => result = PackageRunner.Run("Package1.dtsx", new
                {
                    Source_ConnectionString = sourceDatabase.ConnectionString.ToSsisCompatibleConnectionString(),
                    Dest_ConnectionString = destDatabase.ConnectionString.ToSsisCompatibleConnectionString(),                    
                }));
 ```
 The test then runs the `Package1.dtsx` SSIS package and importantly sets the source and destination connection strings to our newly created databases in the steps above.  This allows us to test the logic inside our SSIS package with our test databases.
 
 ```
 "Then the package should execute successfully"
                ._(() => result.Should().BeTrue());
 ```
 The test then asserts that the package executed successfully by checking the return value of our `PackageRunner.Run` method.  In this case I am just using a bool to return whether or not the package completed without any errors. 
 
 
  ```
 "And the products table in the destination database should contain the row from the source database"
               ._(() => destDatabase.AssertTable().Products.ContainsExactlyOneRowMatching(
                   new {
                       ProductCode = 1,
                       ShippingWeight= 2f,
                       ShippingLength= 3f,
                       ShippingWidth= 4f,
                       ShippingHeight= 5f,
                       UnitCost= 6f,
                       PerOrder= 2
                        }
                   ));
 ```
 The last step asserts that the data has been written correctly to the destination database.  The `ContainsExactlyOneRowMatching` method uses [Simple.Data](http://simplefx.org/simpledata/docs/) under the covers to turn the anonymous object that is passed in, into a sql where clause that runs against the target table in the test database.  The target table is determined by the name you use after `AssertTable()` as `AssertTable()` returns dynamic.  In this case we use Products so the where clause is built against the products table.  This makes the asserting of the fact the data exists very fast as it is being done at the database level.  The whole line of assertion code was designed to be completely readable.  If you just read the line of code you can tell exactly what it is doing.
 
 Once the test finishes all of the databases are dropped when the `testServer` are disposed so the test does not leave anything behind.
 
 By building on this technique it is possible to write automated tests around every aspect of your SSIS project which is something that often lacks test coverage and can be the cause of a lot of pain.
