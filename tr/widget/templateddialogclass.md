# TemplatedDialog Sınıfı

TemplatedWidget'ın alt sınıfı olan TemplatedDialog, jQuery UI Dialog plugin'i kullanarak sayfa içi modal dialog'lar oluşturmamızı sağlar.

## TemplatedDialog'un Bağlandığı Element

Diğer widget türlerinden farklı olarak, TemplatedDialog, bağlanacağı HTML elementini kendi oluşturur. Bu element, `document.body`'e eklenen boş bir `DIV`'dir. Aşağıdaki `Q.NewBodyDiv()` statik fonksiyonu ile elde edilir:

```cs
public static jQueryObject NewBodyDiv()
{
    return J("<div/>").AppendTo(Document.Body);
}
```
Bu nedenle TemplatedDialog constructor'ı, diğer widget'lardan farklı olarak element parametresi almaz:

```cs
public abstract class TemplatedDialog<TOptions> : TemplatedWidget<TOptions>, IDialog
    where TOptions : class, new()
{
    protected TemplatedDialog(TOptions opt = null)
        : base(Q.NewBodyDiv(), opt)
    {
    	InitDialog();
    	// ...
    }
```

Ancak otomatik oluşturulan bu element'e `this.Element` özelliği üzerinden erişilebilir (Widget'ın alt sınıfı olduğunu için).


## TemplatedDialog ve jQuery UI Dialog Widget'ı

TemplatedDialog, boş bir `DIV` oluşturup, `document.body`'ye ekledikten hemen sonra, bu `DIV` üzerinde bir jQuery UI dialog widget'ı oluşturur.

```cs
protected virtual void InitDialog()
{
	this.element.Dialog(this.GetDialogOptions());
    // ...
}
```

jQuery dialog widget'ına geçirilen opsiyonlar GetDialogOptions() metodu çağrılarak elde edilir:

```cs
protected virtual DialogOptions GetDialogOptions()
{
    DialogOptions opt = new DialogOptions();

    opt.DialogClass = "s-Dialog " + "s-" + this.GetType().Name;
    opt.Width = 920;
    opt.AutoOpen = false;
    opt.Resizable = false;
    opt.Modal = true;
    opt.Position = new string[] { "center", "center" };

    return opt;
}
```

Görüldüğü gibi varsayılan opsiyonlarda `s-Dialog` CSS sınıfı, 920px genişlik, dialog'un oluşturulduğu gibi otomatik açılmaması, resize edilememesi, modal olarak açılması (yani açıldığında altındaki arayüzle işlem yapılamaması) ve ekranın ortasında çıkması bulunuyor.

Bu opsiyonlar istenirse `GetDialogOptions()` ezilerek değiştirilebilir.

## TemplatedDialog ve HTML Yapısı

Aşağıdaki gibi basit bir dialog nesnesi tanımlayalım:

```cs
public class MyDialog : TemplatedDialog
{
	// ...
}
```

Diyelim ki sayfamızın ilk HTML body kodu aşağıdaki gibi olsun:

```html
<body>
	<div id="SiteMainDiv">
    	Some content
    </div>
</body>
```

Bu dialog'tan bir adet oluşturalım:
```cs
new MyDialog();
```

Bu satırı çalıştırdığımız anda HTML body içeriği şuna benzer şekilde değişecektir:

```html
<body>
	<div id="SiteMainDiv">
    	Some content
    </div>
    <div class="ui-dialog ui-widget ... s-Dialog s-MyDialog" style="display: none">
    	<div class="ui-dialog-titlebar">
        	Dialog title
        </div>
    	<div id="ui-id-8" class="ui-dialog-content">
        	Dialog content
        </div>
    </div>
</body>
```

Burada `ui-` öneki ile başlayan sınıflar jQuery UI dialog widget'ı tarafından eklendi.

Bizim daha önce `Q.NewBodyDiv()` ile oluşturup `document.body`'ye eklediğimiz element, iç kısımdaki `ui-dialog-content` sınıfına sahip `DIV` dir (`ui-id-8` ID'li).

Bu şaşırtıcı gelebilir, çünkü DIV'imiz `document.body`'nin hemen altında olmalıydı.

Ancak jQuery UI dialog, bizim `DIV` imizi dialog'a çevirirken, özel başka bir `DIV` ile çevreler (wraps). Bu dış `DIV`, `.ui-dialog` sınıfına sahiptir.

Dialog'un stillenmesini kolaylaştırabilmek için, `s-MyDialog` CSS sınıfı, widget'ın bağlı olduğu, bizim oluşturduğumuz element yerine, onu içeren üst `DIV`'e eklenir.

Hatırlanırsa daha önce, bu CSS sınıfının, widget'in elementine tanımlandığını söylemiştik. TemplatedDialog bu noktada da diğer widget lardan biraz farklı davranır.

## TemplatedDialog - Dialog İşlemleri / Element İlişkisi

Daha önce belirttiğimiz gibi `ui-dialog` sınıfına sahip olan dış çerçeve sadece görünüm amaçlıdır ve widget üzerindeki tüm `.dialog()` işlemleri, iç DIV üzerinden gerçekleştirilir.

Yani, örneğin dialog u açmak (görüntülemek) istediğinizde, aşağıdaki gibi bir javascript kodu yazarsanız:

```js
$('.ui-dialog').dialog('open');
```

yanlış bir işlem yapmış olursunuz ve şöyle bir hata alırsınız:

```js
Error: cannot call methods on dialog prior to initialization; attempted to call method 'open'
```

Doğrusu şu şekildedir:
```js
$('.ui-dialog-content').dialog('open');
```

## TemplatedDialog - Dialog'u Görüntüleme (Açma)

GetDialogOptions() fonksiyonunda görüldüğü gibi, oluşturulan dialog otomatik olarak açılmaz (`AutoOpen = false`).

Oluşturduğunuz bir dialog'u açmak istediğinizde aşağıdaki gibi yazabilirsiniz:

```cs
var dialog = new MyDialog();
dialog.Element.Dialog().Open();
```

TemplatedDialog'un kısayolu ile:
```cs
var dialog = new MyDialog();
dialog.DialogOpen();
```

## TemplatedDialog - Dialog'u Gizleme (Kapatma)

Açık olan bir dialog'u kapamak istediğinizde aşağıdaki gibi yazabilirsiniz:
```cs
var dialog = new MyDialog();
dialog.DialogOpen();
// ...
dialog.DialogClose();
```

TemplatedDialog kapatıldığı anda bağlı olduğu HTML element'i DOM'dan ayrılır ve kaynakları boşaltılır. Bu şekilde hafıza sızıntısı (memory leak) oluşması engellenir. Dolayısıyla *kapattığınız bir dialog'u tekrar açamazsınız, yeni bir tane oluşturmalısınız*.
