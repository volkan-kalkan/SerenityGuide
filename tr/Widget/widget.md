# Widget Sınıfı

Serenity Script Arayüzü (*Serenity.UI*) katmanında, bileşen sınıfları (*component / control*) için, *jQuery UI*'daki *Widget Factory*'e benzeyen, ancak C#'a daha uygun tasarlanan bir yapı baz alınmıştır.

> Aşağıdaki adreslerde jQuery UI widget yapısı ile ilgili bilgi alınabilir:
>
> http://learn.jquery.com/jquery-ui/widget-factory/
>
> http://msdn.microsoft.com/en-us/library/hh404085.aspx

Widget, bir HTML elementi üzerinde oluşturulan, bu elemente ek özellikler (*behaviour*) kazandıran bir nesnedir.

Örneğin IntegerEditor widget'ı, bir INPUT elementine uygulanır ve bu input üzerinde sayı girişini kolaylaştırıp, girilen verinin geçerli bir tamsayı olmasını doğrular (validasyon).

Benzer şekilde bir Toolbar widget'ı, bir DIV elementine uygulanır ve bu DIV'i butonları olan bir araç çubuğuna dönüştür (burada DIV sadece yer tutucu olarak görev görür).

## Widget Sınıf Diyagramı
![Widget Class Diagram][1]

## Örnek Bir Widget

Bir DIV'e her tıklandığında font büyüklüğünü arttıran basit bir widget yapalım:

```cs
namespace MySamples
{
    public class MyCoolWidget : Widget
    {
        private int fontSize = 10;

        public MyCoolWidget(jQueryObject div)
            : base(div)
        {
            div.Click(e => {
                fontSize++;
                this.Element.Css("font-size", fontSize + "pt");
            });
        }
    }
}
```

```html
<div id="SomeDiv">Örnek Metin</div>
```

Aşağıdaki gibi bu objeyi, HTML elementi üzerinde oluşturabiliriz:

```cs
var div = jQuery.Select("#SomeDiv");
new MyCoolWidget(div);
```

## Widget Sınıfı Metod ve Özellikleri
```cs
public abstract class Widget : ScriptContext
{
    private static int NextWidgetNumber = 0;

    protected Widget(jQueryObject element);
    public virtual void Destroy();

    protected virtual void OnInit();
    protected virtual void AddCssClass();

    public jQueryObject Element { get; }
    public string WidgetName { get; }
    public string UniqueName { get; }
}
```

### *Widget.Element* Özelliği (Property)

Widget sınıfından türeyen nesneler, kendilerinin bağlı olduğu HTML elementine, `Element` özelliği üzerinden erişebilirler.

```cs
public jQueryObject Element { get; }
```

Bu özellik jQueryObject tipindedir ve widget oluşturulurken belirtilen elementi içerir. Örneğimizde, click olayı içerisinde elementimize (`div`), `this.Element` şeklinde eriştik.

### HTML Elementi ve Widget *CSS Sınıfı* İlişkisi

Bir HTML elementi üzerinde bir widget oluşturduğunuzda, HTML elementinde bazı düzenlemeler yapılır.

İlk olarak HTML elementine, üzerinde oluşan widget'ın tip adına göre bir CSS sınıfı (class) eklenir.

Örneğimizdeki `#SomeDiv` ID'sine sahip `DIV` elementine `.s-MyCoolWidget` sınıfı eklenmiş oldu.

Yani DIV aşağıdaki gibi oldu:

```html
<div id="SomeDiv" class="s-MyCoolWidget">Örnek Metin</div>
```

CSS sınıfı, widget sınıf adının başına `s-` getirilerek elde edilir. (Widget.AddCssClass metodu override edilerek bu isimlendirme değiştirilebilir)

### Widget CSS Sınıfı İle HTML elementinı Stillendirme

Bu CSS sınıfı sayesinde, dilersek HTML elementini, üzerinde oluşturulan widget tipine göre CSS tarafında stillendirebiliriz.

```css
.s-MyCoolWidget {
	background-color: red;
}
```

### HTML elementindan *jQuery.Data* İle Widget'a Erişim

Ayrıca elementimızın data özelliğine de üzerine eklenen widget ile ilgili bir bilgi eklendi. Chrome konsolunda bu veriyi aşağıdaki gibi görebiliriz:

```js
> $('#SomeDiv').data()

> Object { MySamples_MyCoolWidget: $MySamples_MyCoolWidget }
```

Yani, data özelliği üzerinden, bir elemente daha önce bağladığımız herhangi bir Widget'e ulaşmamız mümkün.

```cs
var myWidget = (MyCoolWidget)(J("#SomeDiv").GetDataValue('MySamples_MyCoolWidget'));
```

### WidgetExtensions.GetWidget Uzantı Metodu

Biraz uzun ve karışık gözüken bu kod parçası yerine, Serenity kısayolu kullanılabilir:

```cs
var myWidget = J("#SomeDiv").GetWidget<MyCoolWidget>();
```

Bu kod parçası eğer widget varsa döndürecek, yoksa hata verecektir:

```text
Element has no widget of type 'MySamples_MyCoolWidget'!
```

### WidgetExtensions.TryGetWidget Uzantı Metodu

Eğer element'in üzerinde widget var mı kontrol etmek isterseniz:

```cs
var myWidget = J("#SomeDiv").TryGetWidget<MyCoolWidget>();
```

`TryGetWidget`, eğer widget element'e bağlanmışsa bulup döndürür, yoksa hata vermek yerine `null` sonucunu verir.


### Aynı HTML elementinde Birden Çok Widget

Bir HTML elementine, aynı sınıftan tek bir widget bağlanabilir.

Aynı sınıftan ikinci bir widget oluşturmaya çalıştığınızda aşağıdaki gibi bir hata alırsınız:

```text
Element already has widget 'MySamples_MyCoolWidget'
```

Farklı sınıflardan, aynı elemente birden fazla Widget eklenebilir, tabi yaptıkları işlemlerin birbiriyle çakışmaması kaydıyla.

### *UniqueName* Özelliği

Oluşturulan her bir Widget, otomatik olarak eşsiz (unique) bir isim alır (`MySamples_MyCoolWidget3`) gibi.

`this.UniqueName` üzerinden erişilebilen bu ismi, Widget'ın bağlandığı eleman ya da içinde oluşturacağınız alt HTML elemanları için ID prefix'i (ön eki) olarak kullanarak, sayfa içindeki diğer Widget'larla çakışmayacak eşsiz ID'ler elde edebilirsiniz.

Ayrıca, event handler'larınızı jQuery üzerinden bu isim sınıfıyla bağlayıp (bind), daha sonra yine aynı sınıfla kaldırarak (unbind), aynı eleman üzerine atanmış diğer event handler'ları etkilemekten korunabilirsiniz:

```cs
jQuery("body").Bind("click." + this.UniqueName, delegate { ... });

...

jQUery("body").Unbind("click." + this.UniqueName);
```

### Widget.Destroy Metodu

Bir element üzerinde oluşturduğunuz widget'ı, bazen yoketmek (free) isteyebilirsiniz.

Bunun için Widget sınıfı `Destroy` metodunu sunar.

Destroy metodunun varsayılan implementasyonunda, element üzerindeki bu widget tarafından atanmış event handler'lar temizlenir (uniqueName class'ı kullanılarak) ve widget için eklenmiş CSS sınıfı (`.s-WidgetClass`) kaldırılır.

Özel Widget sınıflarında, Destroy metodu override edilerek, HTML elementi üzerinde yapılan değişiklikler geri alınabilir.

Destroy metodu, HTML elementi DOM'dan ayrıldığında (detach) otomatik olarak çağrılır. Manuel de çağrılması mümkündür.

Destroy işlemi sağlıklı bir şekilde yapılmadığı taktirde, bazı browser'larda hafıza sızıntısı (memory leak) oluşması sözkonusu olabilir.

# Widget&lt;TOptions&gt; Generic Sınıfı

Eğer widget'iniz oluşturulurken, üzerinde oluşturulacağı HTML elementi dışında, bazı ek opsiyonlara ihtiyaç duyuyorsa, `Widget<TOptions>` sınıfını baz alabilirsiniz.

Bu opsiyonlara, constructor'dan direk erişilebileceğiniz gibi, diğer sınıf metodları içinden, `this.options` özelliği aracılığıyla da erişilebilirsiniz.

```cs
public abstract class Widget<TOptions> : Widget
    where TOptions: class, new()
{
    protected Widget(jQueryObject element, TOptions opt = null) { ... }
    protected readonly TOptions options;
}
```

  [1]: WidgetHierarchy.png
