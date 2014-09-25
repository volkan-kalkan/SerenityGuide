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

# LocalCache Static Class

[**namespace**: *Serenity*] - [**assembly**: *Serenity.Core*]

A static class that contains shortcuts to work easier with the *local cache* (current ILocalCache provider).


A default implementation (`Serenity.Caching.HttpRuntimeCache`) that uses `System.Web.Cache` exists in `Serenity.Web` assembly.


```cs
namespace Serenity
{
    public static class LocalCache
    {
        public static void AddToCacheWithExpiration(string cacheKey,
	        object value, TimeSpan expiration);

        public static TItem Get<TItem>(string cacheKey,
	        TimeSpan expiration, Func<TItem> loader)
	        where TItem : class;

        public static TItem TryGet<TItem>(string cacheKey)
            where TItem : class;

        public static void Remove(string cacheKey);
        public static Cache GetCache();
        public static void Reset();
    }
}

```

### LocalCache.AddToCacheWithExpiration

Use this method to add a value to local cache with an expiration time.

```cs
LocalCache.AddToCacheWithExpiration("someKey", "someValue",
	TimeSpan.FromMinutes(5));
```

This method, in its default implementation, uses HttpRuntime.Cache.Insert method.
> Avoid HttpRuntime.Cache.Add method, as it doesn't update value if there is already a key with same key in the cache, and it doesn't even raise an error so you won't notice anything. A mere engineering gem from ASP.NET)

Values are added to cache with absolute expiration (thus they expire at a certain time, not sliding expiration).

### LocalCache.Remove Function

Daha önce önbelleğe eklenmiş bir değeri önbellekten siler (yoksa hata vermez).

```cs
LocalCache.Remove("someKey");
```

### LocalCache.Reset Fonksiyonu

Önbellekteki tüm key leri siler. `HttpRuntime.Cache` tüm verileri silmek için bir metod sunmadığından tek tek key'ler bulunup Remove ile silinir. Bu yüzden gerekmedikçe (genelde testlerde gerekir) kullanmayınız.

### LocalCache.GetCache Fonksiyonu

HttpRuntime.Cache'e erişim sağlar.

### LocalCache.TryGet`<TItem>` Fonksiyonu

HttpRuntime.Cache'den verilen anahtara sahip değeri, istenen tiple okumaya çalışır.

Anahtar önbellekte yoksa ya da tipi TItem değilse null döndürür.

TItem referans tipinde olmalıdır.

### LocalCache.Get`<TItem>` Fonksiyonu

HttpRuntime.Cache'den verilen anahtara sahip değeri okur.

![LocalCache.Get Flow Diagram](img/local_cache_get.png?v4)

* Eğer değer DBNull.Value ise sonuç null olarak döndürülür.

* Eğer değer var ise tipinin TItem olduğu kontrol edilerek değer döndürülür. Tipi farklıysa hata oluşur. (TryGet'ten farklı olarak)

* Eğer değer önbellekte bulunamazsa, loader (yükleyici) fonksiyonu çağrılıp ilk değer üretilir.
	* Loader'ın ürettiği değer null ise önbelleğe bu değer DBNull.Value olarak saklanır.
	* Değer null değil ise verilen expiration süresi ile önbelleğe eklenir.

TItem referans tipinde olmalıdır.

Değerler önbelleğe AbsoluteExpiration ile (belli bir tarihte expire olacak şekilde) eklenir. SlidingExpiration kullanılmaz.

### Kullanıcı Profili Önbellekleme Örneği

Bir sitemizde birkaç sorgu ile oluşturan kullanıcı profil sayfası bulunsun. Bu sayfayı üretmek için gerekli bilgileri veritabanından yükleyip CachedProfile adlı bir objeye doldurabildiğimizi varsayalım:

```cs
public class CachedProfile
{
	public string Name { get; set; }
	public List<CachedFriend> Friends { get; set; }
	public List<CachedAlbum> Albums { get; set; }
	...
}
```

Veritabanından CachedProfile objesi üreten fonksiyonumuzu LocalCache.Get metodu ile kullanarak, profillerin önbelleklenmesini çok basit bir şekilde sağlayabiliriz:

```cs
public CachedProfile GetProfile(int profileId)
{
	return LocalCache.Get<CachedProfile>(
		cacheKey: "Profile:" + profileId,
		expiration: TimeSpan.FromHours(1),
		loader: delegate {
			using (var connection = new SqlConnection("..."))
			{
				// load profile by profileId from DB
			}
		}
	);
}
```
