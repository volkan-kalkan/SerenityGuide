# ScriptContext Sınıfı
C#, Javascript'ten farklı olarak bir sınıfa bağlı olmayan global fonksiyonlara izin vermez.

Bu nedenle jQuery'nin `$` fonksiyonunu Saltarelle ile aynı basitlikte kullanamayız. `$('#SomeElementId)` gibi basit bir javascript ifadesinin, Saltarelle'deki karşılığı `jQuery.Select("#SomeElementId")` dir.

Bunun için *ScriptContext* sınıfı geliştirilmiştir.

```cs
public class ScriptContext
{
    [InlineCode("$({p})")]
    protected static jQueryObject J(object p);
    [InlineCode("$({p}, {context})")]
    protected static jQueryObject J(object p, object context);
}
```

`$`, C#'da geçerli bir metod ismi (identifier) olmadığından onun yerine `J` kullanılmıştır. ScriptContext'ten türeyen sınıflarda, jQuery() fonksiyonuna kısaca `J()` ile erişilebilir.

```cs

public class SampleClass : ScriptContext
{
    public void SomeMethod()
    {
        J("#SomeElementId").AddClass("abc");
    }
}
```
