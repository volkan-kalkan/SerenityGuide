# TemplatedWidget Sınıfı

Geliştireceğiniz Widget, hedef aldığı HTML elementi içinde, aşağıdaki gibi, karmaşık bir HTML içerik oluşturacak ise, bu markup'ı direk Widget metodlarında üretmeniz çorba (spagetti) koda yol açabilir. Ayrıca markup kod ile üretildiğinden, ihtiyaca göre özelleştirilmesi de zorlaşır.

```cs
public class MyComplexWidget : Widget
{
	public MyComplexWidget(jQueryObject div)
    	: base(div)
    {
		var toolbar = J("<div>")
        	.Attribute("id", this.uniqueName + "_MyToolbar")
            .AppendTo(div);

        var table = J("<table>")
        	.AddClass("myTable")
            .Attribute("id", this.uniqueName + "_MyTable")
            .AppendTo(div);

        var header = J("<thead/>").AppendTo(table);
        var body = J("<tbody>").AppendTo(table);
        ...
        ...
        ...
    }
}
```

Bu sorun HTML şablonu kullanarak çözülebilir. Örneğin aşağıdaki kodu HTML sayfa kodumuza ekleyelim:

```html
<script id="Template_MyComplexWidget" type="text/html">
<div id="~_MyToolbar">
</div>
<table id="~_MyTable">
	<thead><tr><th>Name</th><th>Surname</th>...</tr></thead>
    <tbody>...</tbody>
</table>
</script>
```

Burada `SCRIPT` tag'i kullandık, ancak tipini `"text/html"` yaparak, browser tarafından script olarak algılanmasını engelledik.

TemplatedWidget sınıfını baz alarak önceki kodu şu hale getirelim:

```cs
public class MyComplexWidget : TemplatedWidget
{
	public MyComplexWidget(jQueryObject div)
    	: base(div)
    {
    }
}
```

Bu widget'ı aşağıdaki gibi bir HTML elementi üzerinde oluşturduğunuzda:

```html
<div id="SampleElement">
</div>
```

Şöyle bir görünüm elde edersiniz:

```html
<div id="SampleElement">
    <div id="MySamples_MyComplexWidget1_MyToolbar">
    </div>
    <table id="MySamples_MyComplexWidget1_MyTable">
        <thead><tr><th>Name</th><th>Surname</th>...</tr></thead>
        <tbody>...</tbody>
    </table>
</div>
```

TemplatedWidget, otomatik olarak sizin sınıfınız için hazırladığınız şablonu bulur ve HTML elementine uygular.

## TemplatedWidget ID Üretimi

Dikkat ederseniz şablonumuzda alt elementlerin ID'lerini `~_MyToolbar`, `~_MyTable` şeklinde yazdık.

Ancak şablon HTML elementine uygulandığında üretilen ID'ler sırasıyla **MySamples_MyComplexWidget1_MyToolbar** ve `MySamples_MyComplexWidget1_MyTable` oldu.

TemplatedWidget, şablondaki `~_` öneklerini, Widget'ın eşsiz ismi (`uniqueName`) ve alt çizgi "_" ile değiştirir (`_` de içeren bu ön eke `this.idPrefix` alanı üzerinden erişilebilir).

Bu yolla, aynı Widget şablonu, aynı sayfada birden fazla HTML elementinde kullanılsa bile, ID'lerin çakışması önlenir.

## TemplatedWidget.ByID Metodu

ID'lerin başlarına TemplateWidget'ın eşsiz ismi getirildiğinden, şablon HTML elementine uygulandıktan sonra, üretilen alt HTML elemanlarına, şablonda belirttiğiniz ID'ler ile direk ulaşamazsınız.

elemanları ararken ID'lerinin başına TemplatedWidget'ın uniqueName'ini ve alt çizgi karakterini (`_`) getirmeniz gerekir:

```cs
public class MyComplexWidget : TemplatedWidget
{
	public MyComplexWidget(jQueryObject div)
    	: base(div)
    {
    	J(this.uniqueName + "_" + "Toolbar").AddClass("some-class");
    }
}
```

Bunun yerine TemplatedWidget'ın ByID metodu kullanılabilir:

```cs
public class MyComplexWidget
{
	public MyComplexWidget(jQueryObject div)
    	: base(div)
    {
    	ByID("Toolbar").AddClass("some-class");
    }
}
```

## TemplatedWidget.GetTemplateName Metodu

Örneğimizde, `MyComplexWidget` sınıfımız kullanacağı şablonu otomatik olarak bulmuştu.

Bunun için TemplatedWidget bir çıkarım yaptı (convention based programming). Sınıfımızın class isminin (`MyComplexWidget`) başına `Template_` önekini getirdi ve bu ID'ye (`Template_MyComplexWidget`) sahip bir `SCRIPT` elementi arayıp, bulduğu elementin içeriğini şablon olarak kabul etti.

Eğer biz şablonumuzun ID'sini `Template_MyComplexWidget` yerine aşağıdaki gibi değiştirseydik:

```html
<script id="TheMyComplexWidgetTemplate" type="text/html">
	...
</script>
```

Bu durumda tarayıcı konsolunda şöyle bir hata alırdık:

```text
Can't locate template for widget 'MyComplexWidget' with name 'Template_MyComplexWidget'!
```

Şablonumuzun ID'sini düzeltebilir, ya da widget'imizin bu ismi kullanmasını sağlayabiliriz:

```cs
public class MyComplexWidget
{
	protected override string GetTemplateName()
    {
    	return "TheMyComplexWidgetTemplate";
    }
}
```

## TemplatedWidget.GetTemplate Metodu

Eğer şablonunuzu başka bir kaynaktan getirmek ya da manuel olarak oluşturmak isterseniz, `GetTemplate` metodunu override edebilirsiniz:

```cs
public class MyCompleWidget
{
	protected override string GetTemplate()
    {
    	return J("#TheMyComplexWidgetTemplate").GetHtml();
    }
}
```


## Q.GetTemplate Metodu ve Sunucu Tarafı Şablonları

TemplatedWidget.GetTemplate metodu varsayılan implementasyonunda, GetTemplateName metodunu çağırıp, bu ID'ye sahip `SCRIPT` elementinin HTML içeriğini şablon olarak döndürür.

Eğer SCRIPT elementi bulunamaz ise, `Q.GetTemplate` metodu bu şablon ismiyle çağrılır.

O da sonuç döndürmez ise şablonun tespit edilemediğine dair hata verilir.

`Q.GetTemplate` metodu ise, sunucu tarafında tanımlanmış şablonları getirir.

Bu şablonlar, `~/Views/Template` ya da `~/Modules` klasörleri ve alt klasörlerinde bulunan `.template.cshtml` ya da `.template.html` uzantılı dosyalardan derlenir.

Örneğin, MyComplexWidget için kullacağınız şablonu, sunucuda `~/Views/Template/SomeFolder/MyComplexWidget.template.cshtml` gibi bir dosyaya aşağıdaki içerikle yazabilirsiniz:

```html
<div id="~_MyToolbar">
</div>
<table id="~_MyTable">
	<thead><tr><th>Name</th><th>Surname</th>...</tr></thead>
    <tbody>...</tbody>
</table>
```

Dosyanın adı ve uzantısı önemlidir, hangi alt klasörde olduğunun bir önemi yoktur.

Bu şekilde sayfanın HTML koduna herhangi bir widget şablonu eklenmesi gerekmez.

Ayrıca bu şablonlar ilk ihtiyaç olduğunda yüklendiğinden (*lazy loading*), ve sunucu ve istemci tarafında önbelleklendiğinden (*cached*), sayfa kodunuz belki de hiç kullanmayacağınız widget'lar için gereksiz yere şişmez. Bu yüzden sayfa içi (inline) SCRIPT şablonları yerine, sunucu tarafı şablonları kullanmanız tavsiye edilir.
