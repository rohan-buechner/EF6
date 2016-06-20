# EF6

Bsg.EF6 is a .Net based framework, extending Microsoft's Entity Framework 6 (Code First) with a specific focus on bulk operations.
The framework is used internally by the .Net teams at [Business Systems Group](http://www.bsg.co.za/) when working with Entity Framework 6. 

### What is its purpose?
The aim of the framework is leverage off the DbContext's standard functionality to automatically generate raw sql from strongly-typed IQueryables
and then combine these parameterised SELECT SQL statements with custom SQL to perform bulk SQL operations (INSERTS, UPDATES, DELETES etc.) in one command, directly on the SQL server without requiring the DbConext to submit multiple requests from the Application Layer.

### What are the advantages to using it?
Using this framework can reduce the work load and memory requirements on the Application Server, reduce the number of (and frequeny of opening and closing) connections required to communicate with the Database Server and significantly reduce the network traffic between the two. Whilst at the same time still leveraging most of the usual benefits of a traditional ORM. 

### What does this actually mean?
It means you can run bulk update, insert and delete SQL statements directly on the Database server, but still make use of a ORM.
I.e. you get the performance of SQL and the usability of the ORM.

### How is this different from other Bulk EF frameworks?
For some parts, its very similar especially with regards to the BulkAdd, BulkDelete and BulkUpdate type functionality.
Its more unique value proposition comes from the SelectAndAdd and SelectAndUpdate methods, 
which allow for bulk (many) and unique (per row) updates and inserts without sending any additional data across the network (assuming all the data needed for the update or insert already resides within the Database) 

## BulkDelete & BulkUpdate

![BulkAdd](docs/bulkdeleteupdate.png)

#### Overview
Delete or update multiple entities in a single call based on a Predicate Expression (lamda) or IQueryable of that type. 

#### When To Use
For **BulkDelete**, when you need to delete many (but not necessarily all) records without reseeding the primary keys back to 0 and are able to define a predicate type expression or provide a IQueryable collection of that type. 

If only 1 record is required to be deleted consider using the non bulk methods.

If all records are to be deleted consider using the **Truncate** methods.

For **BulkUpdate**, when you need to update a selection (via a predicate expression or an IQueryable) of records with a constant update (i.e. same values for everyrow to be updated).

If only 1 record is required to be updated consider using the standard EF change tracking functionality.

If many records are to be updated uniquely (same properties but different values depending on the record to be updated) and these new values can be found from data already residing within the database, then consider using the **SelectAndUpdate** methods.

#### Examples

Delete with IQueryable
```csharp
var recordsToDelete = someRepo.FindAll(e => !e.IsActive);
someRepo.BulkDelete(recordsToDelete);
```

Delete with Predicate Expression
```csharp
someRepo.BulkDelete(e => !e.IsActive);
```

Update with IQueryable
```csharp
var recordsToDelete = someRepo.FindAll(e => !e.IsActive);
someRepo.BulkDelete(recordsToDelete);
```

Update with Predicate Expression
```csharp
var recordsToUpdate = someRepo.FindAll(e => !e.IsActive);
someRepo.BulkUpdate(recordsToUpdate, e => new SomeEntity { IsActive = true, Comment = "Reactivated" });
```

## BulkAdd

![BulkAdd](docs/bulkadd.png)

#### Overview
Insert multiple entities in large batches rather than on a row by row basis. Behind the scenes these methods make use of DataTables and SQL Bulk Copy and related .Net libraries. 
 
#### When To Use
When multiple records are required to be inserted and the data currently resides only on the application server side (i.e. not within the database).

If only 1 record is required to be inserted and / or the autogenerated Id is required immediately from this inserted entity, then use the standard EF change tracking functionality.

If many records are required to be created, but the data to create, compose and/ or aggregate these new records is all available within the database, then consider using the **SelectAndAdd** methods.   

#### Examples
```csharp
var newEntities = new List<SomeEntity>();

...code to create entities...

someRepo.BulkAdd(newEntities);
```

## Truncate

![BulkAdd](docs/truncate.png)

#### Overview
Remove all entities from a table and reset the autogenerated Id back to 0. 

#### When To Use
Because the SQL Trunacte command won't work on tables with foreign key references, 2 truncate methods are provided.

If only some records are required to be removed consider using some of the other Bulk and Non Bulk delete methods.

#### Examples

```csharp
using (var transaction = contextSession.StartNewTransaction())
{
    someRepo.Truncate(transaction);
    someOtherRepo.TruncateWithForeignKeys(transaction);
    transaction.Commit();
}
```

## BulkSelectAndAdd & BulkSelectAndUpdate

![BulkAdd](docs/bulkselectand.png)

#### Overview
Used to insert or update many entities using data already available within the database. This prevents having to move large quantities between the application and database servers.
Note that constant data can still be passed from the application server to the database server as part of these methods.

*Note: Refer to the limitations further down on this page regarding the need for wrappers and specific ordering of properties when using the SelectAndX methods.*

#### When To Use
For **BulkSelectAndAdd**, when many new records need to be created and these records can be composed of data that can be obtained from simple or complex queries directly on the database server.

For **BulkSelectAndUpdate**, when many new records need to be updated uniquely and this unique data (per record) can be composed of data that can be obtained from simple or complex queries directly on the database server.

#### Examples
```csharp
var insertQuery =   someOtherEntityRepo
                    .FindAll(e => e.SomeProperty.StartsWith("Something"))
                    .GroupBy(e => e.SomeProperty)
                    .Select(g => new SomeEntityWrapper
                    {
                        Name = g.Key,
                        IsActive = g.Sum(e => e.SomeValue - e.SomeOtherValue) > SomeConstant
                    });

someRepo.BulkSelectAndAdd(insertQuery);
```

```csharp
var updateQuery =   from someOtherEntity in someOtherIQueryable
                    join someEntity in someIQueryable on someOtherEntity.SomeKey equals someEntity.SomeKey
                    select new SomeEntityWrapper
                    {
                        SomeKey = someEntity.SomeKey,
                        IsActive = someOtherEntity.SomeProperty > SomeContant
                    };

someRepo.BulkSelectAndUpdate(updateQuery);
```

## Traditional Entity Framework

![BulkAdd](docs/traditional.png)

#### Overview
Standard Entity Framework ORM functionality with change tracking, lazy loading, eager loading etc. can be obtained through the Non Bulk methods mostly found on the base repo *IRepository*.

Methods include **Find**X, **Count**X, **Delete**X, **Add**X (where X can refer to *One* or *All* as well as *Tracked* overloads) and updates which are are done implicitly through the change tracking mechanisms, without the need for a dedicated **Update**X method. 

#### When To Use
The important thing to note about these methods are that they can result in many individual queries to the database server,
which is perfectly fine for individual entities changes, but can also introduce significant delays when dealing with more than a few hundred records at a time.
Whereas the bulk methods should be able to handle hundreds of thousands if not millions of records in a fraction of the time.  

#### Examples
FindAll
```csharp
var activeEntities = someRepo.FindAll(e => e.IsActive).ToList();
```

FindAll (with dynamic Sorting, Ordering and Paging)
```csharp
var activeEntities = someRepo.FindAll(e => e.IsActive, "SomeProperty", "DESC", 100, 2).ToList();
```

CountAll
```csharp
var noOfActiveEntities = someRepo.CountAll(e => e.IsActive);
```

DeleteAll
```csharp
someRepo.DeleteAll(e => e.IsActive);
```

Update
```csharp
var item = someRepo.FindOneTracked(e => e.Id == SomeId);
item.SomeProperty = "Changed Property";
contextSession.CommitChanges();
```


## Getting Started
The topics below should give a brief introduction to the more important components of the framework.
Alternatively the Bsg.Ef6.Tests project can also be seen as a mini POC showing how to setup the IOC containers, how to create custom DbContexts, how to create domain entities, configurations, context repos, entities repos and most importantly how to use the various methods of the abstract bulk repos defined in the framework.

*Note: the that all the Contexts (Primary, Secondary), Entities (Alpha, Beta, One, Two etc.) and all their properties within the test project are completely arbitary ito their names.*

### DI & Built-In Services
The Bsg.EF6 framework includes a number of required Services, most of which can be overridden with custom implementations.
All the services have a default concrete implementation corresponding to the same name (excluding the Interface "I" prefix).
The services rely on the Constructor Injection convention for their DI implementations. 
Unless otherwise stated below, a transient lifecycle (or equivelent) is recommened.

See code the *TestIocBootstrapper* class in the Bsg.Ef6.Tests project, for an example of implementing the framework's services in an IOC (in this case Autofac). 

#### Table Mapping Services####
**ITableMappingsCacheService** - Stores the cached values for table mappings (various mappings between DB Columns and Poco properties) for each context. This must be populated at system startup prior to calling any repo methods. This ideally should be a singleton service i.e. Singleton lifecycle.  

**ITableMappingsFactory** - Builds a *ContextTableMappings* object for a specific context - used by the *ITableMappingService* below. Potentailly not required (see *ITableMappingService*).

**ITableMappingService** - Facade across the ITableMappingsCacheService and ITableMappingsFactory service to build and cache every discovered implementation of *Ef6Context*. This should be called manually after the IOC container has been setup.
Alternatively *ITableMappingsFactory* and *ITableMappingService* can be ignored as long as the *ITableMappingsCacheService* has been populated (for every Ef6Context) at startup via some other custom code.

#### Timeout Services####
**ITimeoutCacheService** - Stores the cached values for various timeouts for each context. This must be populated at system startup prior to calling any repo methods. This ideally should be a singleton service i.e. Singleton lifecycle.

**ITimeoutFactory** - Builds a *ContextTimeouts* object for a specific context - used by the *ITimeoutService* below. Potentailly not required (see *ITimeoutService*).

**ITimeoutService** - Facade across the *ITimeoutCacheService* and *ITimeoutCacheFactory* service to build and cache every found discovered implementation of *Ef6Context*. This should be called manually after the IOC container has been setup.
Alternatively *ITimeoutCacheFactory* and *ITimeoutService* can be ignored as long as the *ITimeoutCacheService* has been populated (for every Ef6Context) at startup via some other custom code.

#### Context Services####
**IDbContextFactory** - Builds DbContexts (and can also bind pre-generated EF Views at startup).

**IDbContextSession** - Wrapper around the instance of *Ef6Context* and thus controls its creation and destruction when required. There is generally 1 instance per *Ef6Context* implementation per request (or equivalent scope) to be shared amongst all Repository instances related to that *Ef6Conext* Scoped. Thus a Scoped lifecycle (or equivalent) is required.

**IContextService** - Facade across the IDbContextFactory in order to bind pre-generated EF Views for every *Ef6Context* implementation, in order to speed up the first actual query. This is usually called once at the application startup.

#### Repository Services####
**IGenericRepository** - Is a downstream repository which extends all of *IRepository*, *IBulkInsertRepository* and *IBulkEndbaleRepository* and thus includes all the Bulk and Non-Bulk repo functionality. 
It will be specific to a *Ef6Context* type and an *IEntity* type linked to that context.  
This would normally be regsitered as an open generic in order to prevent having to specifiy each *IEntity* type manually when resgistring types on the IOC container.
All Repositories (including the *IGenericRepository*) wrap around the scoped IDbContextSession which in turn manages the specific instance of Ef6Context.
Alternatively, context specific repos or even entity specific repos can be created and registered by extending off of the IGenericRepository and when doing so, *IGenericRepository* can be ignored.

#### Utility Services ####
**IDatabaseConnectionFactory** - Builds *SqlConnections*

**IConfigurationService** - Retrieves app settings and connection strings from app.config.

**IBulkInserterFactory** - Builds a *BulkInserter* for a specific *IEntity*


### Ef6Context
Extends off the standard *DbContext* and contains extended functionality to: 

- build the *TableMapping* for the specific context (discovered from within the *StoreItemCollection*)
- discover and register all *EntityTypeConfigurations* relating to itself
- expose the generic *IDbSet*, to be used by the various repos

### Repository Pattern


#### Generic Repository
*TODO*
#### Context Repository
*TODO*
#### Entity Repository
*TODO* 
### IWrapperEntity
*TODO*
### EntityTypeConfiguration
*TODO*
### Transactions
*TODO*
### app.config
*TODO*

## Limitations
### Code First
*TODO*
### SelectAndX : Projections using IWrapperEntity
*TODO*
### SelectAndX : Projections and order of properties
*TODO*
### Primary Keys
*TODO*
### Collections
*TODO*
### SqlServer
*TODO*
### Parallel Operations in same container scope
*TODO*
### Nuget Package
*TODO*

