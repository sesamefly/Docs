---
title: Custom storage providers for ASP.NET Core Identity | Microsoft Docs
author: ardalis
description: How to configure custom storage providers for ASP.NET Core Identity.
keywords: ASP.NET Core, Identity, custom storage providers
ms.author: riande
manager: wpickett
ms.date: 05/24/2017
ms.topic: article
ms.assetid: b2ace545-ecf6-4664-b31e-b65bd4a6b025
ms.technology: aspnet
ms.prod: asp.net-core
uid: security/authentication/identity-custom-storage-providers
---
# Custom storage providers for ASP.NET Core Identity

By [Steve Smith](http://ardalis.com)

ASP.NET Core Identity is an extensible system which enables you to create a custom storage provider and connect it to your app. This topic describes how to create a customized storage provider for ASP.NET Core Identity. It covers the important concepts for creating your own storage provider, but is not a step-by-step walkthrough.

[View or download sample from GitHub](https://github.com/aspnet/Docs/tree/master/aspnetcore/security/authentication/identity/sample).

## Introduction

By default, the ASP.NET Core Identity system stores user information in a SQL Server database using Entity Framework Core. For many apps, this approach works well. However, you may prefer to use a different persistence mechanism. For example:

* You use [Azure Table Storage](https://docs.microsoft.com/azure/storage/) or another data store.
* Your database tables have a different structure. 
* You may wish to use a different data access approach, such as [Dapper](https://github.com/StackExchange/Dapper). 

In each of these cases, you can write a customized provider for your storage mechanism and plug that provider into your app.

ASP.NET Core Identity is included in application templates in Visual Studio with the "Individual User Accounts" option.

When using the dotnet CLI, add `-au Individual`:

```
dotnet new mvc -au Individual
dotnet new webapi -au Individual
```

## The ASP.NET Core Identity architecture

ASP.NET Core Identity consists of classes called managers and stores. *Managers* are high-level classes which an app developer uses to perform operations, such as creating a Identity user. *Stores* are lower-level classes that specify how entities, such as users and roles, are persisted. Stores follow the [repository pattern](http://deviq.com/repository-pattern/) and are closely coupled with the persistence mechanism. Managers are decoupled from stores, which means you can replace the persistence mechanism without changing your application code (except for configuration).

The following diagram shows how a web app interacts with the managers, while stores interact with the data access layer.

![ASP.NET Core Apps work with Managers (for example, `UserManager`, `RoleManager`). Managers work with Stores (for example, `UserStore`) which communicate with a Data Source using a library like Entity Framework Core.](identity-custom-storage-providers/_static/identity-architecture-diagram.png)

To create a custom storage provider, create the data source, the data access layer, and the store classes that interact with this data access layer (the green and grey boxes in the diagram above). You don't need to customize the managers or your app code that interacts with them (the blue boxes above).

When creating a new instance of `UserManager` or `RoleManager` you provide the type of the user class and pass an instance of the store class as an argument. This approach enables you to plug your customized classes into ASP.NET Core. 

[Reconfigure app to use new storage provider](#reconfigure-app-to-use-new-storage-provider) shows how to instantiate `UserManager` and `RoleManager` with a customized store.

## ASP.NET Core Identity stores data types

To implement a custom storage provider, it's helpful to understand the types of data used with ASP.NET Core Identity. This helps you decide which features are relevant to your app. The ASP.NET Core stores data types are detailed in the following sections:

### Users

Registered users of your web site. The `User` store includes the following fields:
<!--  Just like english code is not allowed, ditto for data structures. Can't you just include the User class with some comments? How is this useful to creating a custom store? You need to do something like like the list below of the most important fields - but you need to point to the Users model.-->
* user Id and user name. 
* A hashed password if users log in with credentials that are specific to your site (rather than using credentials from an external site like Facebook).
* A security stamp to indicate whether anything has changed in the user credentials. 
* email address, phone number, whether two factor authentication is enabled, the current number of failed logins, and whether an account has been locked.

### User Claims

<!-- Link to this in Identity repo -->
A set of statements (or claims) about the user that represent the user's identity. Can enable greater expression of the user's identity than can be achieved through roles.

### User Logins

<!-- Link to this in Identity repo -->
Information about the external authentication provider (like Facebook or a Microsoft account) to use when logging in a user.

### Roles
<!-- Link to this in Identity repo -->
Authorization groups for your site. Includes the role Id and role name (like "Admin" or "Employee").

## The data access layer

This topic assumes you are familiar with the persistence mechanism that you are going to use and how to create entities for that mechanism. This topic does not provide details about how to create the repositories or data access classes; it provides some suggestions about design decisions when working with ASP.NET Core Identity.

You have a lot of freedom when designing the data access layer for a customized store provider. You only need to create persistence mechanisms for features that you intend to use in your app. For example, if you are not using roles in your app, you do not need to create storage for roles or user role associations. Your technology and existing infrastructure may require a structure that is very different from the default implementation of ASP.NET Core Identity. In your data access layer, you provide the logic to work with the structure of your storage implementation.

The data access layer provides the logic to save the data from ASP.NET Core Identity to a data source. The data access layer for your customized storage provider might include the following classes to store user and role information.

### Context class
<!-- Link to default version in Identity repo and how about link to your custom version? -->
Encapsulates the information to connect to your persistence mechanism and execute queries. Several data classes require an instance of this class. ~You initialize store classes with an instance of this class.~ <!-- Does saying that in English provide any benefit to showing it in code?  Keep it if you think it's valuable, but it should be obvious once you write a custom provider. -->

### User Storage
<!-- Link to default version in Identity repo and how about link to your custom version? -->
Stores and retrieves user information (such as user name and password hash).

### Role Storage
<!-- Link to default version in Identity repo and how about link to your custom version? -->
Stores and retrieves role information (such as the role name).	

### UserClaims Storage
<!-- Link to default version in Identity repo and how about link to your custom version? -->
Stores and retrieves user claim information (such as the claim type and value).

### UserLogins Storage
<!-- Link to default version in Identity repo and how about link to your custom version? -->
Stores and retrieves user login information (such as an external authentication provider).	

### UserRole Storage
<!-- Link to default version in Identity repo and how about link to your custom version? -->
Stores and retrieves which roles are assigned to which users.

**TIP:** Only implement the classes you intend to use in your app.

In the data access classes, provide code to perform data operations for your persistence mechanism. For example, within a custom provider, you might have the following code to create a new user in the *store* class:

[!code-csharp[Main](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/CustomUserStore.cs?name=createuser&highlight=7)]

The implementation logic for creating the user is in the ``_usersTable.CreateAsync`` method, shown below.

## Customize the user class

When implementing a storage provider, create a user class which is equivalent to the [`IdentityUser` class](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnet.identity.corecompat.identityuser).

At a minimum, your user class must include an `Id` and a `UserName` property.

The `IdentityUser` class defines the properties that the ``UserManager`` calls when performing requested operations. The default type of the `Id` property is a string, but you can inherit from `IdentityUser<TKey, TUserClaim, TUserRole, TUserLogin, TUserToken> and specify a different type. The framework expects the storage implementation to handle data type conversions.

## Customize the user store

Create a `UserStore` class that provides the methods for all data operations on the user. This class is equivalent to the [UserStore<TUser>](https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.entityframeworkcore.userstore-1) class. In your `UserStore` class, implement `IUserStore<TUser>` and the optional interfaces required. You select which optional interfaces to implement based on on the functionality provided in your app.

### Optional interfaces

- IUserRoleStore https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserrolestore-1
- IUserClaimStore https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserclaimstore-1
- IUserPasswordStore https://docs.microsoft.com/aspnet/core/api/microsoft.aspnetcore.identity.iuserpasswordstore-1
- IUserSecurityStampStore <!-- make these all links and remove /en-us/ -->
- IUserEmailStore
- IPhoneNumberStore
- IQueryableUserStore
- IUserLoginStore
- IUserTwoFactorStore
- IUserLockoutStore

The optional interfaces inherit from `IUserStore`.

The following image shows the methods in a simple user store class. The `TUser` generic parameter takes the type of your user class.
<!-- Might be better to provide a link to your class - that image doesn't do anything for me. ANd I don't see  The `TUser` generic parameter - so how is that helpful at this point? -->

![A high level view of a custom user store.](identity-custom-storage-providers/_static/custom-user-store.png)

Within the `UserStore` class, you use the data access classes that you created to perform operations. These are passed in using dependency injection. For example, in the SQL Server with Dapper implementation, the `UserStore` class has the `CreateAsync` method which uses an instance of `DapperUsersTable` to insert a new record:

[!code-csharp[Main](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/DapperUsersTable.cs?name=createuser&highlight=7)]

### Interfaces to implement when customizing user store

- **IUserStore**  
 The [IUserStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613278(v=vs.108).aspx) interface is the only interface you must implement in the user store. It defines methods for creating, updating, deleting, and retrieving users.
- **IUserClaimStore**  
 The [IUserClaimStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613265(v=vs.108).aspx) interface defines the methods you implement to enable user claims. It contains methods for adding, removing and retrieving user claims.
- **IUserLoginStore**  
 The [IUserLoginStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613272(v=vs.108).aspx) defines the methods you implement to enable external authentication providers. It contains methods for adding, removing and retrieving user logins, and a method for retrieving a user based on the login information.
- **IUserRoleStore**  
 The [IUserRoleStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613276(v=vs.108).aspx) interface defines the methods implement to map a user to a role. It contains methods to add, remove, and retrieve a user's roles, and a method to check if a user is assigned to a role.
- **IUserPasswordStore**  
 The [IUserPasswordStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613273(v=vs.108).aspx) interface defines the methods implement to persist hashed passwords. It contains methods for getting and setting the hashed password, and a method that indicates whether the user has set a password.
- **IUserSecurityStampStore**  
 The [IUserSecurityStampStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613277(v=vs.108).aspx) interface defines the methods implement to use a security stamp for indicating whether the user's account information has changed. This stamp is updated when a user changes the password, or adds or removes logins. It contains methods for getting and setting the security stamp.
- **IUserTwoFactorStore**  
 The [IUserTwoFactorStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613279(v=vs.108).aspx) interface defines the methods implement to support two factor authentication. It contains methods for getting and setting whether two factor authentication is enabled for a user.
- **IUserPhoneNumberStore**  
 The [IUserPhoneNumberStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613275(v=vs.108).aspx) interface defines the methods implement to store user phone numbers. It contains methods for getting and setting the phone number and whether the phone number is confirmed.
- **IUserEmailStore**  
 The [IUserEmailStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613143(v=vs.108).aspx) interface defines the methods implement to store user email addresses. It contains methods for getting and setting the email address and whether the email is confirmed.
- **IUserLockoutStore**  
 The [IUserLockoutStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613271(v=vs.108).aspx) interface defines the methods implement to store information about locking an account.<!-- sentence way to long. Use a list or better yet a summary. It contains methods for failed attempts and lockout. You don't need an exhaustive list. --> It contains methods for getting the current number of failed access attempts, getting and setting whether the account can be locked, getting and setting the lock out end date, incrementing the number of failed attempts, and resetting the number of failed attempts.
- **IQueryableUserStore**  
 The [IQueryableUserStore&lt;TUser&gt;](https://msdn.microsoft.com/library/dn613267(v=vs.108).aspx) interface defines the members implement to provide a queryable user store.

You implement only the interfaces that are needed in your app. For example:

```csharp
public class UserStore : IUserStore<IdentityUser>,
                         IUserClaimStore<IdentityUser>,
                         IUserLoginStore<IdentityUser>,
                         IUserRoleStore<IdentityUser>,
                         IUserPasswordStore<IdentityUser>,
                         IUserSecurityStampStore<IdentityUser>
{
    // interface implementations not shown
}
```

### IdentityUserClaim, IdentityUserLogin, and IdentityUserRole

The ``Microsoft.AspNet.Identity.EntityFramework`` namespace contains implementations of the [IdentityUserClaim](https://msdn.microsoft.com/library/dn613250(v=vs.108).aspx), [IdentityUserLogin](https://msdn.microsoft.com/library/dn613251(v=vs.108).aspx), and [IdentityUserRole](https://msdn.microsoft.com/library/dn613252(v=vs.108).aspx) classes. If you are using these features, you may want to create your own versions of these classes and define the properties for your app. However, sometimes it is more efficient to not load these entities into memory when performing basic operations (such as adding or removing a user's claim). Instead, the backend store classes can execute these operations directly on the data source. For example, the ``UserStore.GetClaimsAsync`` method can call the ``userClaimTable.FindByUserId(user.Id)`` method to execute a query on that table directly and return a list of claims.

## Customize the role class

When implementing a role storage provider, you can create a custom role type. It need not implement a particular interface, but it must have an `Id` and typically it will have a `Name` property.

The following is an example role class:

[!code-csharp[Main](identity-custom-storage-providers/sample/CustomIdentityProviderSample/CustomProvider/ApplicationRole.cs)]

## Customize the role store

You can create a ``RoleStore`` class that provides the methods for all data operations on roles. This class is equivalent to the  <!--make all these links to the Identity Repo -->``RoleStore<TRole>`` class. In the RoleStore class, you implement the ``IRoleStore<TRole>`` and optionally the ``IQueryableRoleStore<TRole>`` interface.

The following class view diagram shows a role store class. The ``TRole`` generic parameter takes the type of the role class.
<!-- I don't see how that view is helpful - wouldn't the default implementation of that class be more useful? -->

![A high level view of a custom role store.](identity-custom-storage-providers/_static/custom-role-store.png)

- **IRoleStore&lt;TRole&gt;**  
 The [IRoleStore](https://msdn.microsoft.com/library/dn468195.aspx) interface defines the methods to implement in the role store class. It contains methods for creating, updating, deleting and retrieving roles.
- **RoleStore&lt;TRole&gt;**  
 To customize `RoleStore`, create a class that implements the `IRoleStore` interface. Implement this class only when you use roles in your application. The constructor that takes a parameter named *database* of type `ExampleDatabase` is one example of how to pass in the data access class. <!--  What constructor? In your sample? IF so, say that and point to it.-->In the MySQL implementation <!-- where is that? Can you point to it? -->, the constructor takes a parameter of type `MySQLDatabase`.  

## Reconfigure app to use new storage provider

Once you have implemented a storage provider, you configure your app to use it. If your app used the default provider, replace it with your custom provider.

1. Remove the `Microsoft.AspNetCore.EntityFramework.Identity' NuGet package.
1. If the storage provider resides in a separate project or package, add a reference to it.
1. Replace all references to `Microsoft.AspNetCore.EntityFramework.Identity' with a using statement for the namespace of your storage provider.
1. In the ``ConfigureServices`` method, change the `AddIdentity` method to use your custom types. You can create your own extension methods for this purpose.<!-- see (link to custom extension methods)-->
1. If you are using Roles, update the `RoleManager` to use your RoleStore class.
1. Update the connection string and credentials to your app's configuration.

Example:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // Add identity types
    services.AddIdentity<ApplicationUser, ApplicationRole>()
        .AddDefaultTokenProviders();

    // Identity Services
    services.AddTransient<IUserStore<ApplicationUser>, CustomUserStore>();
    services.AddTransient<IRoleStore<ApplicationRole>, CustomRoleStore>();
    string connectionString = Configuration.GetConnectionString("DefaultConnection");
    services.AddTransient<SqlConnection>(e => new SqlConnection(connectionString));
    services.AddTransient<DapperUsersTable>();

    // additional configuration
}
```

## References

- [Custom Storage Providers for ASP.NET Identity](https://docs.microsoft.com/aspnet/identity/overview/extensibility/overview-of-custom-storage-providers-for-aspnet-identity)