# LocalCache Statik Sınıfı

Bu statik sınıf, `HttpRunTime.Cache` ile çalışmak için kısayollar ve yardımcı fonksiyonlar sunar.

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

Önbelleğe bir değeri belli bir süre kalmak üzere eklemek istediğinizde bu fonksiyonu kullanabilirsiniz.

```cs
LocalCache.AddToCacheWithExpiration("someKey", "someValue", 
	TimeSpan.FromMinutes(5));
```

Bu fonksiyon, HttpRuntime.Cache.Insert metodunu kullanır (HttpRuntime.Cache.Add metodunu hiçbir zaman kullanmayınız, bu yordam eğer cache te belirtilen anahtar var ise değerini güncellemez ve bir hata da vermez, Microsoft'tan bir mühendislik harikası)

Değerler önbelleğe AbsoluteExpiration (belli bir tarihte expire olacak şekilde) ile eklenir.

### LocalCache.Remove Fonksiyonu

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