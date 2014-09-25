# User Authorization & Authentication

Serenity uses some abstractions to work with your application's own user authentication and authorization mechanism.

```cs
Serenity.Abstractions.IAuthenticationService
Serenity.Abstractions.IAuthorizationService
Serenity.Abstractions.IPermissionService
Serenity.Abstractions.IUserRetrieveService
```

As Serenity doesn't have default implementation for these abstractions, you should provide some implementation for them, using dependency registration system.

For example, Serenity Basic Application sample registers them in its SiteInitialization.ApplicationStart method like below:

```cs
var registrar = Dependency.Resolve<IDependencyRegistrar>();

registrar.RegisterInstance<IAuthorizationService>(
	new Administration.AuthorizationService());

registrar.RegisterInstance<IAuthenticationService>(
	new Administration.AuthenticationService());

registrar.RegisterInstance<IPermissionService>(
	new Administration.PermissionService());

registrar.RegisterInstance<IUserRetrieveService>(
	new Administration.UserRetrieveService());
```

You might want to have a look at these sample implementations before writing your own.

## IAuthenticationService Interface

[**namespace**: *Serenity.Abstractions*, **assembly**: *Serenity.Core*]

```cs
public interface IAuthenticationService
{
    bool Validate(ref string username, string password);
}
```

This is the service that you usually call from your login page to check if entered credentials are correct. Your implementation should return true if username/password pair matches.

A dummy authentication service could be written like this:

```cs
public class DummyAuthenticationService : IAuthenticationService
{
    public bool Validate(ref string username, string password)
    {
        return username == password;
    }
}
```

This service returns true, if username is equal to specified password (just for demo).

First parameter is passed by reference for you to change username to its actual representation in database before logging in. For example, the user might have entered uppercase `JOE` in login form, but actual username in database could be `Joe`.  This is not a requirement, but if your database is case sensitive, you might have problems during login or later phases.

You might register this service from global.asax.cs / SiteInitialization.ApplicationStart like:

```cs
protected void Application_Start(object sender, EventArgs e)
{
	Dependency.Resolve<IDependencyRegistrar>()
		.RegisterInstance(new DummyAuthenticationService());
}
```

And use it somewhere in your login form:

```cs
void DoLogin(string username, string password)
{
	if (Dependency.Resolve<IAuthenticationService>()
			.Validate(ref username, password))
	{
		// FormsAuthentication.SetAuthenticationTicket etc.
	}
}
```

# IAuthorizationService Interface

[**namespace**: *Serenity.Abstractions*, **assembly**: *Serenity.Core*]

This is the interface that Serenity checks through to see if there is a logged user in current request.

```cs
public interface IAuthorizationService
{
    bool IsLoggedIn { get; }
    string Username { get; }
}
```

Some basic implementation for web applications could be:

```cs
using Serenity;
using Serenity.Abstractions;

public class MyAuthorizationService : IAuthorizationService
{
    public bool IsLoggedIn
    {
        get { return !string.IsNullOrEmpty(Username); }
    }

    public string Username
    {
        get { return WebSecurityHelper.HttpContextUsername; }
    }
}

/// ...

void Application_Start(object sender, EventArgs e)
{
	Dependency.Resolve<IDependencyRegistrar>()
		.RegisterInstance(new MyAuthorizationService());
}
```


# IPermissionService
[**namespace**: *Serenity.Abstractions*, **assembly**: *Serenity.Core*]

A permission is some grant to do some action (entering a page, calling a certain service). In Serenity, permissions are some string keys that are assigned to individual users (similar to ASP.NET roles).

For example, if we say some user has `Administration` permission, this user can see navigation links that requires this permission, or call services that require the same.

> You can also use composite permission keys like `ApplicationID:PermissionID` (for example `Orders:Create`), but Serenity doesn't care about application ID, nor permission ID, it only uses the composite permission key as a whole.


```cs
public interface IPermissionService
{
    bool HasPermission(string permission);
}
```

You might have a table like...

```sql
CREATE TABLE UserPermissions (
	UserID int,
    Permission nvarchar(20)
}
```

and query it to implement this interface.

A simpler sample for applications where there is a `admin` user who is the only one that has the permission `Administration` could be:

```cs
using Serenity;
using Serenity.Abstractions;

public class DummyPermissionService : IPermissionService
{
    public bool HasPermission(string permission)
    {
        if (Authorization.Username == "admin")
            return true;

        if (permission == "Administration")
            return false;

        return true;
    }
}
```

## IUserDefinition Interface

[**namespace**: *Serenity*, **assembly**: *Serenity.Core*]

Most applications store some common information about a user, like id, display name (nick / fullname), email address etc. Serenity provides a basic interface to access this information in an application independent manner.

```cs
public interface IUserDefinition
{
    string Id { get; }
    string Username { get; }
    string DisplayName { get; }
    string Email { get; }
    Int16 IsActive { get;  }
}
```

Your application should provide a class that implements this interface, but not all of this information is required by Serenity itself. Id, Username and IsActive properties are minimum required.

`Id` can be an integer, string or GUID that uniquely identifies a user.

`Username` should be a unique username, but you can use e-mail addresses as username too.

`IsActive` should return 1 for active users, -1 for deleted users (if you don't delete users from database), and 0 for temporarily disabled (account locked) users.

`DisplayName` and `Email` are optional and currently not used by Serenity itself, though your application may require them.

## IUserRetrieveService Interface
[**namespace**: *Serenity.Abstractions*, **assembly**: *Serenity.Core*]

When Serenity needs to access IUserDefinition object for a given user name or user ID, it uses this interface.

```cs
public interface IUserRetrieveService
{
    IUserDefinition ById(string id);
    IUserDefinition ByUsername(string username);
}
```

In your implementation, it is a good idea to cache user definition objects, as a common WEB application might use this interface repeatedly for same user.

Serenity Basic Application sample has an implementation like below:

```cs
public class UserRetrieveService : IUserRetrieveService
{
    private static MyRow.RowFields fld { get { return MyRow.Fields; } }

    private UserDefinition GetFirst(IDbConnection connection, BaseCriteria criteria)
    {
        var user = connection.TrySingle<Entities.UserRow>(criteria);
        if (user != null)
            return new UserDefinition
            {
                UserId = user.UserId.Value,
                Username = user.Username,
                //...
            };

        return null;
    }

    public IUserDefinition ById(string id)
    {
        if (id.IsEmptyOrNull())
            return null;

        return TwoLevelCache.Get<UserDefinition>("UserByID_" + id, CacheExpiration.Never, CacheExpiration.OneDay, fld.GenerationKey, () =>
        {
            using (var connection = SqlConnections.NewByKey("Default"))
                return GetFirst(connection,
                    new Criteria(fld.UserId) == id.TryParseID32().Value);
        });
    }

    public IUserDefinition ByUsername(string username)
    {
        if (username.IsEmptyOrNull())
            return null;

        return TwoLevelCache.Get<UserDefinition>("UserByName_" + username, CacheExpiration.Never, CacheExpiration.OneDay, fld.GenerationKey, () =>
        {
            using (var connection = SqlConnections.NewByKey("Default"))
                return GetFirst(connection, new Criteria(fld.Username) == username);
        });
    }
}
```
