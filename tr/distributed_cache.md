# Distributed Cache

Web uygulamaları, aynı anda yüzlerce, binlerce belki de daha fazla kullanıcıya hizmet vermek durumundadır. Gerekli önlemleri almazsanız, böyle bir yük altında siteniz cevap veremez duruma gelebilir. 

Mesela sitenizin ana sayfasında en son 10 haberi gösteriyorsunuz. Aynı dakika içerisinde bu sayfaya binlerce giriş olduğunu düşünelim. Her giriş yapan kullanıcı için SQL den bu haberleri çeken yordamınız çalışacaktır. 

```sql
SELECT TOP 10 Title, NewsDate, Subject, Body FROM News ORDER BY NewsDate DESC
```

Ana sayfamızda başka hiçbirşey olmadığını düşünsek bile, dakikada 10000 ziyaret alan bir site, saniyede 150 civarında SQL sorgusu çalıştırıyor demektir.

Bu sorgular, kullanıcıdan kullanıcıya değişmediğinden (hep son 10 haber), sonuçları SQL sunucunuz tarafında otomatik olarak bir miktar cache'lenebilir. Ancak sorgu sonuçları SQL sunucudan WEB sunucunuza transfer olurken değerli ağ bant genişliğinizden harcar. Bu transfer belli bir süre aldığından, ve bu süre boyunca bağlantınız açık kaldığından, SQL sunucunuz sorguya anında yanıt verse dahi, sonuçları almanız o kadar hızlı olmayacaktır. Bu süre haber içeriklerinizin boyutuna göre değişebilir. Ayrıca açık SQL bağlantılarının bir sınırı vardır (connection pool limiti), ve bu sayıya ulaşıldığında bağlantılar kuyrukta beklemeye ve birbirlerini bloklamaya başlarlar.

Haberlerin her saniye değişmediğini düşünerek, bunları WEB sunucumuzun hafızasında 5 dakika kadar saklayabiliriz.

Yani haber listesini SQL'den çektikten hemen sonra önbelleğe (cache) atıyoruz ve 5 dk boyunca ana sayfaya giren tüm kullanıcılar için haber listesini önbellekten getiriyoruz:

```cs
public List<News> GetNews() 
{
    var news = HttpRuntime.Cache["News"] as List<News>;
    if (news == null) 
    {
        using (var connection = new SqlConnection("......"))
        {
            news = connection.Query<News>("
                   SELECT TOP 10 Title, NewsDate, Subject, Body 
                   FROM News 
                   ORDER BY NewsDate DESC")
               .ToList();

            HttpRuntime.Cache.Insert("News", ..., 
                TimeSpan.FromMinutes(5), ....);
        }
    }

    return news;
}
```

Saniyede 150 civarında sorgudan, saniyede 1'in altında sorguya indik (300 sn de 1 sorgu, sn de 1/300 sorgu).

Tabi bu haberler her kullanıcı için HTML e çevrilmek zorunda. Bir adım daha ileri giderek, haberlerin HTML e çevrilmiş halini de önbellekleyebilirdik. 

Tüm bu önbelleklenmiş bilgiler direk sunucu hafızasında, erişimin en hızlı olduğu noktada saklanıyor.

Şimdi, Facebook gibi bir sitemiz olduğunu, milyonlarca kullanıcı profil sayfamız olduğunu düşünün. Belli kişilerin profil sayfaları dakikada yüzlerce, binlerce ziyaret alıyor olabilir. Bir kişinin profil sayfasını üretmek için birden fazla SQL sorgusuna ihtiyacımız olacaktır (arkadaşlar, resim sayısı, albüm isimleri, kişisel bilgiler, son durumu vs.) . Sonuçta kişi profilini güncellemedikçe sayfasında gözükenler üç aşağı beş yukarı statiktir. Dolayısıyla profil sayfasının anlık bir durumunu yine 5 dk, 1 saat gibi bir süre önbellekte saklayabiliriz.

Ancak bu yeterli olmayabilir. Dediğimiz gibi yüzmilyonlarca profil, yüzmilyonlarca kullanıcı söz konusu. Bu kullanıcılar profil sayfalarına bakmaktan çok daha fazlasını yapıyor olabilirler. Tek bir sunucu kesinlikle yeterli olmayacaktır. Dünyanın farklı yerlerinde binlerce sunucuya ihtiyacımız olacak. 

Bu sunucuların herbiri aynı kişinin, örneğin bir ünlünün profilini (adı Çok Ünlü Kişi olsun) kendi yerel hafızalarında önbelleklemiş olabilir. Bunun için hepsi ilk erişimde (5 dk da bir), SQL sorguları ile Çok Ünlü Kişi'nin profil bilgilerini oluşturabilir.  Çok Ünlü Kişi' profilinde bir değişiklik yaptığı anda, tüm bu sunucular bir şekilde önbelleklemelerini yenilemeli, ki bu işlem üç aşağı beş yukarı aynı saniye içinde olacaktır. Kullanıcı başına SQL sunucuda oluşan yükten, sunucu başına oluşan yüke geçiş yaptık.

Aslında sunuculardan biri Çok Ünlü Kişi'nin profilini SQL sunucudan çekip oluşturduğunda, diğer sunucularımız da bu bilgiden faydalanabilir. Ancak her sunucu kendi yerel hafızasında bu bilgileri önbellekliyor. Dolayısıyla basit bir şekilde bu bilgiye erişim mümkün değil.

Aslında elimizde herkesin erişebildiği şöyle bir ortak hafıza olsa:

Bilgi Anahtarı           |Değer
-------------------------|----------------
Profil:CokUnluKisi       |<div>Çok Ünlü Kişi<div>....
Profil:BaskaBirKisi      |<div>Başka Bir Kişi<div>....
...                      |...
...                      |...
Profil:AliVeli           |<div>Ali Veli<div>....

Bütün sunucular da eğer yerel önbelleklerinde bir kişinin profil bilgileri yoksa, bu bilgileri SQL den getirmeye çalışmadan önce bu ortak hafızaya baksalar, sunucu başına SQL sunucu üzerinde oluşturulacak yükten büyük ölçüde kurtulmuş olurduk:


```cs
public CachedProfileInformation GetProfile(string profileID) 
{
    var profile = HttpRuntime.Cache["Profil:" + profileID] 
        as CachedProfileInformation;
    
    if (profile == null) 
    {
        profile = DistributedCache.Get<CachedProfileInformation>(
            "Profil:" + profileID);

        if (profile == null)
        {
            using (var connection = new SqlConnection("......"))
            {
                profile = GetProfileFromDBWithSomeSQLQueries(profileID)
                    profile, TimeSpan.FromMinutes(5));
                    
                DistributedCache.Set("Profil:" + profileID, profile, 
	                TimeSpan.FromHours(1));
            }
        }
    }

    return news;
}
```

İşte piyasada Memcached, Couchbase, Redis gibi değişik versiyonlarını bulabileceğiniz distributed cache ya da diğer adıyla NoSQL veritabanları bu amaçla kullanılır. 

```
DİKKAT! Eğer tek bir sunucunuz varsa dağıtık cache kullanmak fayda yerine zarar verebilir. Sadece ileride birden fazla WEB sunucu kullanmak gibi bir planınınız varsa, ileride uğraşmamak için distributed cache kullanabilirsiniz, ama
performans düşecektir.
```

Distributed cache'i (dağıtık önbellek) basit bir şekilde uzak bilgisayarlarda konumlandırılmış bir dictionary (sözlük) olarak düşünebilirsiniz. Bu sunucular, (key, value) ikililerini (anahtar/değer) hafızalarında saklarlar ve bu bilgilere mümkün olduğunca hızlı erişmeye imkan tanırlar.

Önbelleklenen veri çok fazla olduğunda, tek bir bilgisayarın hafızası bütün key/value ikililerini tutmak için yeterli gelmeyebilir. Bu durumda memcached gibi sunucu tipleri clustering yaparak, verileri sunucular arasında dağıtırlar. Basit bir örnek vermek gerekirse, anahtarların ilk harfine göre clustering yapabilirdiniz. Yani A ile başlayan veriler bir sunucu grubunda, B ile başlayanlar başka bir grupta vs. gibi. (Tabi gerçek uygulama bu kadar basit değil, genelde key'in hash değerine göre bir sınıflandırma yapılıyor)

Tüm NoSQL sunucu tipleri, hemen hemen aynı arayüzü sunarlar. En temel iki işlem şu anahtar değeri için şu değeri sakla veş u anahtar değerine karşılık gelen değeri getirdir.

Serenity'de dağıtık önbellek desteğini, belli bir sunucu türüne bağlı kalmadan sağlayabilmek için IDistributedCache arayüzü tanımlanmıştır:

```cs
public interface IDistributedCache
{
    long Increment(string key, int amount = 1);
    TValue Get<TValue>(string key);
    void Set<TValue>(string key, TValue value);
    void Set<TValue>(string key, TValue value, DateTime expiresAt);
}
```


Set metodunun key ve value parametresi alan ilk overload'ı dağıtık önbellekte bir değeri belirlemek için kullanılır. 

```cs
IoC.Resolve<IDistributedCache>().Set("someKey", "someValue");
```

Daha sonra bu değeri Get metodu ile geri okuyabiliriz:

```cs
var value = IoC.Resolve<IDistrubutedCache().Get<string>("someKey") // someValue
```

Bir değeri cache'te belli bir süre saklamak istersek Set'in expiresAt parametresi alan overload'ını kullanabiliriz.

```cs
IoC.Resolve<IDistributedCache>().Set("someKey", "someValue", 
    DateTime.Now.AddMinutes(10));
```

Normalde dağıtık mimaride yapılan işlemler atomik değildir ve transaction mantığı bulunmaz. Aynı anahtar birden fazla sunucu tarafından aynı anda değiştirilebilir, birbirlerinin yazdıkları değerleri ezebilirler. Sunucular arasında unique bir counter, mesela ID üretmek ve bunu distributed cache üzerinden senkronize etmek (aynı ID nin iki defa kullanılmasını kesin olarak engellemek) istedik diyelim:

```cs
int BirSonrakiIDDegeriniAl() 
{
    var lastID = IoC.Resolve<IDistributedCache>().Get("LastID");
    IoC.Resolve<IDistributedCache>().Set("LastID", lastID + 1);
    return lastID;
}
```

Böyle bir kod istediğiniz gibi çalışmayacaktır. Sizin `LastID` değerini okumanız (get) ile `LastID` değerini sunucuda arttırmak için yazmanız (set) arasında geçen sürede başka bir sunucuda değeri okuyup yazmış olabilir. Dolayısıyla iki sunucuda aynı ID yi kullanabilir.

Bunun için Increment metodunu kullanabilirsiniz:

```cs
int BirSonrakiIDDegeriniAl() 
{
    return IoC.Resolve<IDistributedCache>().Increment("LastID");
}
```

Increment fonksiyonu thread senkronizasyonunda kullanılan Interlocked.Increment gibi çalışır ve dağıtık sunucudaki anahtarın değerini diğer istekleri bloklayarak arttırıp, size arttırılmış ID değerini döndürür. Dolayısıyla iki WEB sunucu bu metodu aynı anda çalıştırsa bile dönen ID değerleri birbirinden farklı olacaktır.

## DistributedCacheEmulator

Distributed cache kullanmanız şu an için gerekmediğinde, ancak kodlarınızı ileride dağıtık önbellek ile çalışabilecek şekilde yazmak istediğinizde DistributedCacheEmulator kullanabilirsiniz. Birim testleri için de DistributedCacheEmulator'ten faydalanılabilir.

DistributedCacheEmulator, IDistributedCache arayüzünü, hafızada bir dictionary ile thread-safe olarak sağlar. 

IDistributedCache arayüzüne ek olarak testlerde önbelleği temizleyebilmek için `Reset` metodu sunar:

```cs
((DistributedCache)(IoC.Resolve<IDistributedCache>())).Reset();
```

Distributed cache'in kullanılabilmesi için öncelikle IoC container'e kayıtlanması (register) gerekir, bu işlem genellikle SiteInitialization.cs te yapılır (program başlangıcında çağrılan herhangi bir yerden, ör. global.asax.cs)

```cs
private static void InitIoC()
{
    // ...
    if (ConfigurationManager.AppSettings["DistributedCache"].IsEmptyOrNull())
        IoC.RegisterInstance<IDistributedCache>(new DistributedCacheEmulator());
    else
        //...
        
    // ...
}

```

Bu örnekte konfigürasyon dosyasında DistributedCache ayarları bulunamamışsa (birazdan inceleyeceğiz), IDistributedCache servisi olarak DistributedCacheEmulator'ü kayıt ediyoruz.

## CouchbaseDistributedCache

Couchbase dağıtık bir veritabanıdır. Memcached ile benzer bir erişim arayüze sahiptir (memcached modunda).

Serenity, Couchbase'e özel kütüphanelere bağımlı kalmamak için, bu implementasyonu direk dosyalarında barındırmaz. Ancak `contrib\` altında CouchbaseDistributedCache.cs dosyasını bulabilirsiniz.

IoC.RegisterInstance<IDistributedCache>(new CouchbaseDistributedCache());

Sunucu ayarları konfigürasyon dosyasındaki (web.config / app.config) appSettings\DistributedCache altında JSON formatında bir değerle yapılır.

```xml
<appSettings>
    <add key="DistributedCache" value='{ 
        ServerAddress: "http://111.22.111.97:8091/pools", 
        BucketName: "primary-bucket", 
        KeyPrefix: "", 
        SecondaryServerAddress: "http://111.22.111.96:8091/pools|second-bucket;http://172.16.191.96:8091/pools|default;http://111.22.111.97:8091/pools|third-bucket" }'
    />
```

Buradaki ServerAdress Couchbase sunucu adresi, BucketName bucket adıdır. 

Eğer aynı sunucu / bucket ikilisini birden fazla uygulamada kullanmak isterseniz, key'lerin önüne bir ön-ek getirerek uygulamaların önbelleklenmiş verilerinin birbirlerine karışmamasını sağlanabilir. Bunun için KeyPrefix ayarına örneğin `DEV:`, `TEST:`, `Uygulama1:` gibi bir değer yazılabilir.

Couchbase sunucusu birincil sunucuya ek olarak, bir ya da birden fazla ikincil sunuculara izin vermektedir. Birincil sunucuda yapılan bazı işlemler (sadece silme, yani değeri null a set etme), burada belirtilen ikincil sunucu / bucket'lara da uygulanmaktadır. Yani aşağıdaki kod çalıştırıldığında:

```cs
IoC.Resolve<IDistributedCache>().Set("SomeSharedValue", null);
```

`SomeSharedValue` anahtarlı değer, birinci sunucudan (`111.22.11.97:8091/pools|primary-bucket`) silindiği gibi diğer iki ikincil sunucudan (`http://111.22.111.96:8091/pools|second-bucket, http://111.22.111.97:8091/pools|third-bucket`) da silinir. 

Bu ikincil sunucuların listesini, sunucu adı ve bucket arasına `|` koyup, sunucuları `;` ile ayırarak, `SecondaryServerAddress` içinde girebilirsiniz.

Bu ayarlar konfigürasyon dosyasında bulunamadığı taktirde CouchbaseDistributedCache nesnesi oluşturulurken hata verecektir.

## RedisDistributedCache

Couchbase benzeri bir sunucu olan Redis ile çalışmak için bu nesneden faydalanabilirsiniz. Couchbase'den farklı olarak bu sunucu türünde bucket bulunmamakta (benzeri olan bir db index ayarı var, fakat şimdilik bu desteklenmiyor). İkincil sunucu desteği de verilmemektedir.

Ancak genel olarak Redis daha performanslı bir sunucudur. (StackOverflow tek bir Redis sunucu üzerinde çalışmakta, ve bu sunucunun CPU kullanımının %1-2 seviyesinde olduğunu söylemekteler)

Ayarları aşağıdaki gibi yapılabilir:

```xml
<appSettings>
    <add key="DistributedCache" value="{ 
	    ServerAddress: 'someredisserver:6379', 
	    KeyPrefix: '' 
    }"/>
/>
```

