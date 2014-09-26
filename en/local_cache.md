# ILocalCache Interface
[**namespace**: *Serenity.Abstrations*] - [**assembly**: *Serenity.Core*]

Defines a basic interface to work with the local cache.

The term *local* means that cached items are hold in local memory (thus there is no serialization involved).

```cs
public interface ILocalCache
{
    void Add(string key, object value, TimeSpan expiration);
    TItem Get<TItem>(string key);
    object Remove(string key);
    void RemoveAll();
}
```

> A default implementation of ILocalCache (`Serenity.Caching.HttpRuntimeCache`) that uses `System.Web.Cache` exists in `Serenity.Web` assembly.

### ILocalCache.Add Method

Adds a value to cache with the specified key. If the key already exists in cache, its value is updated.

Items are hold in cache for `expiration` duration. You can specify `TimeSpan.Zero` for items that shouldn't expire automatically.

Values are added to cache with absolute expiration (thus they expire at a certain time, not sliding expiration).

```cs
Dependency.Resolve<ILocalCache>.Add("someKey", "someValue", TimeSpan.FromMinutes(5));
```

This method, in its default implementation, uses HttpRuntime.Cache.Insert method.
> Avoid HttpRuntime.Cache.Add method, as it doesn't update value if there is already a key with same key in the cache, and it doesn't even raise an error so you won't notice anything. A mere engineering gem from ASP.NET)

### ILocalCache.Get`<TItem>` Method

Gets the value corresponding to the specified key in local cache.

If there is no such key in cache, an error may be raised only if TItem is of value type. For reference types returned value is `null`.

If value is not of type `TItem`, an exception is thrown.

You may use `object` as `TItem` parameter to prevent errors in case a value doesn't exist, or not of requested type.

### ILocalCache.Remove Method

Removes the item with specified key from local cache and returns its value.

No errors thrown if there is no value in cache with the specified key, simply `null` is returned.

```cs
Dependency.Resolve<ILocalCache>.Remove("someKey");
```

### ILocalCache.RemoveAll Method

Removes all items from local cache. Avoid using this except for special situations like unit tests, otherwise performance might suffer.

# LocalCache Static Class

[**namespace**: *Serenity*] - [**assembly**: *Serenity.Core*]

A static class that contains shortcuts to work easier with the registered ILocalCache provider.

```cs
public static class LocalCache
{
    public static void Add(string key, object value, TimeSpan expiration);
    public static TItem Get<TItem>(string key, TimeSpan expiration,
        Func<TItem> loader) where TItem : class;
    public static void Remove(string key);
    public static void RemoveAll();
}
```

Add, Remove, and RemoveAll methods are simply shortcuts to corresponding methods in ILocalCache interface, but Get method is a bit different than ILocalCache.Get.

### LocalCache.Get`<TItem>` Method

Gets the value corresponding to the specified key in local cache.

If there is no such key in cache, uses the loader function to produce value, and adds it to cache with the specified key.

![LocalCache.Get Flow Diagram](img/local_cache_get_en.png?v4)

* If the value that exists in cache is DBNull.Value, than null is returned. (This way, if for example a user with an ID doesn't exist in database, repeated querying of database for that ID is prevented)

* If the value exists, but of not type TItem an exception is thrown, otherwise value is returned.

* If the value didn't exist in cache, loader function is called to produce the value (e.g. from database) and...
	* If the value produced by loader function is null, it is stored as DBNull.Value in cache.
	* Otherwise the produced value is added to cache with the specified expiration duration.

#### User Profile Caching Sample

Lets assume we have a profile page in our site that is generated using several queries. We might have a model for this page e.g. UserProfile class that contains all profile data for a user, and a GetProfile method that produces this for a particular user id.

```cs
public class UserProfile
{
	public string Name { get; set; }
	public List<CachedFriend> Friends { get; set; }
	public List<CachedAlbum> Albums { get; set; }
	...
}
```

```cs
public UserProfile GetProfile(int userID)
{
	using (var connection = new SqlConnection("..."))
	{
        // load profile by userID from DB
    }
}
```

By making use of LocalCache.Get method, we could cache this information for one hour easily and avoid DB calls every time this information is needed.

```cs
public UserProfile GetProfile(int userID)
{
	return LocalCache.Get<UserProfile>(
		cacheKey: "UserProfile:" + userID,
		expiration: TimeSpan.FromHours(1),
		loader: delegate {
			using (var connection = new SqlConnection("..."))
			{
				// load profile by userID from DB
			}
		}
	);
}
```
