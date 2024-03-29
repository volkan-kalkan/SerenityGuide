# ScriptContext Class
C#, doesn't support global methods, so jQuery's `$` function can't be used as simply in Saltarelle as it is in Javascript.

A simple expression like `$('#SomeElementId)` in Javascript corresponds to Saltarelle C# code `jQuery.Select("#SomeElementId")`.

As a workaround, *ScriptContext* class can be used:

```cs
public class ScriptContext
{
    [InlineCode("$({p})")]
    protected static jQueryObject J(object p);
    [InlineCode("$({p}, {context})")]
    protected static jQueryObject J(object p, object context);
}
```

As `$` is not a valid method name in C#, `J` is chosen instead. In subclasses of ScriptContext, jQuery.Select() function can be called briefly as `J()`.

```cs

public class SampleClass : ScriptContext
{
    public void SomeMethod()
    {
        J("#SomeElementId").AddClass("abc");
    }
}
```

# Widget Class

Serenity Script UI layer's component classes (control) are based on a system that is similar to *jQuery UI*'s *Widget Factory*, but redesigned for C#.

> You can find more information about jQuery UI widget system here:
>
> http://learn.jquery.com/jquery-ui/widget-factory/
>
> http://msdn.microsoft.com/en-us/library/hh404085.aspx

Widget, is an object that is attached to an HTML element and extends it with some behaviour.

For example, IntegerEditor widget, when attached to an INPUT element, makes it easier to enter numbers in the input and validates that the entered number is a correct integer.

Similarly, a Toolbar widget, when attached to a DIV element, turns it into a toolbar with tool buttons (in this case, DIV acts as a placeholder).

## Widget Class Diagram
![Widget Class Diagram][1]

## A sample Widget

Let's build a widget, that increases a DIV's font size everytime it is clicked:

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
<div id="SomeDiv">Sample Text</div>
```

We can create this widget on an HTML element, like:

```cs
var div = jQuery.Select("#SomeDiv");
new MyCoolWidget(div);
```

## Widget Class Members
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

### *Widget.Element* Property

Classes derived from Widget can get the element, on which they are created, by the `Element` property.

```cs
public jQueryObject Element { get; }
```

This property has type of jQueryObject and returns the element, which is used when the widget is created. In our sample, container DIV element is referenced as `this.Element` in the click handler.

### HTML Element and Widget *CSS Class*

When a widget is created on an HTML element, it does some modifications to the element.

First, the HTML element gets a CSS class, based on the type of the widget.

In our sample, `.s-MyCoolWidget` class is added to the `DIV` with ID `#SomeDiv`.

Thus, after widget creation, the DIV looks similar to this:

```html
<div id="SomeDiv" class="s-MyCoolWidget">Sample Text</div>
```

This CSS class is generated by putting a `s-` prefix in front of the widget class name (it can be changed by overriding Widget.AddCssClass method).

### Styling the HTML Element With Widget CSS Class

Widget CSS class can be used to style the HTML element that the widget is created on.

```css
.s-MyCoolWidget {
	background-color: red;
}
```

### Getting a Widget Reference From an HTML Element with the *jQuery.Data* Function

Along with adding a CSS class, another information about the widget is added to the HTML element, though it is not obvious on markup. This information can be seen by typing following in Chrome console:

```js
> $('#SomeDiv').data()

> Object { MySamples_MyCoolWidget: $MySamples_MyCoolWidget }
```

Thus, it is possible to get a reference to a widget that is attached to an HTML element, using `$.data` function. In C# this can be written as:

```cs
var myWidget = (MyCoolWidget)(J("#SomeDiv").GetDataValue('MySamples_MyCoolWidget'));
```

### WidgetExtensions.GetWidget Extension Method

Instead of the prior line that looks a bit long and complex, a Serenity shortcut can be used:

```cs
var myWidget = J("#SomeDiv").GetWidget<MyCoolWidget>();
```

This piece of code returns the widget if it exists on HTML element, otherwise throws an exception:

```text
Element has no widget of type 'MySamples_MyCoolWidget'!
```

### WidgetExtensions.TryGetWidget Extension Method

TryGetWidget can be used to check if the widget exists, simply returning `null` if it doesn't:

```cs
var myWidget = $('#SomeDiv').TryGetWidget<MyCoolWidget>();
```


### Creating Multiple Widgets on an HTML Element

Only one widget of the same class can be attached to an HTML element.

An attempt to create a secondary widget of the same class on a element throws the following error:

```text
The element already has widget 'MySamples_MyCoolWidget'.
```

Any number of widgets from different classes can be attached to a single element as long as their behaviour doesn't affect each other.

### Widget.*UniqueName* Property

Every widget instance gets a unique name like `MySamples_MyCoolWidget3` automatically, which can be accessed by `this.UniqueName` property.

This unique name is useful as a ID prefix for the HTML element and its descendant elements which may be generated by the widget itself.

It can also be used as an event class for `$.bind` and `$.unbind' methods to attach / detach event handlers without affecting other handlers, which might be attached to the element:

```cs
jQuery("body").Bind("click." + this.UniqueName, delegate { ... });
...

jQUery("body").Unbind("click." + this.UniqueName);
```

### Widget.Destroy Method

Sometimes releasing an attached widget might be required without removing the HTML element itself.

Widget class provides `Destroy` method for the purpose.

In default implementation of the Destroy method, event handlers which are assigned by the widget itself are cleaned (by using UniqueName event class) and its CSS class (`.s-WidgetClass`) is removed from the HTML element.

Custom widget classes might need to override Destroy method to undo changes on HTML element and release resources (though, no need to detach handlers that are attached previously with UniqueName class)

Destroy method is called automatically when the HTML element is detached from the DOM. It can also be called manually.

If destory operation is not performed correctly, memory leaks may occur in some browsers.

# Widget&lt;TOptions&gt; Generic Class

If a widget requires some additional initialization options, it might be derived from the `Widget<TOptions>` class.

The options passed to the constructor can be accessed in class methods through the protected field `options`.

```cs
public abstract class Widget<TOptions> : Widget
    where TOptions: class, new()
{
    protected Widget(jQueryObject element, TOptions opt = null) { ... }
    protected readonly TOptions options;
}
```

#TemplatedWidget Sınıfı

A widget that generates a complicated HTML markup in its constructor or other methods might lead to a class with much spaghetti code that is hard to maintain. Besides, as markup lies in program code, it might be difficult to customize output.

```cs
public class MyComplexWidget : Widget
{
	public MyComplexWidget(jQueryObject div)
    	: base(div)
    {
		var toolbar = J("<div>")
        	.Attribute("id", this.UniqueName + "_MyToolbar")
            .AppendTo(div);

        var table = J("<table>")
        	.AddClass("myTable")
            .Attribute("id", this.UniqueName + "_MyTable")
            .AppendTo(div);

        var header = J("<thead/>").AppendTo(table);
        var body = J("<tbody>").AppendTo(table);
        ...
        ...
        ...
    }
}
```

Such problems can be avoided by using HTML templates. For example, lets add the following template into the HTML page:

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

Here, a `SCRIPT` tag is used, but by specifying its type as `"text/html"`, browser won't recognize it as a real scriptto execute.

By making use of TemplatedWidget, lets rewrite previous spaghetti code block:

```cs
public class MyComplexWidget : TemplatedWidget
{
	public MyComplexWidget(jQueryObject div)
    	: base(div)
    {
    }
}
```

When this widget is created on an HTML element like following:

```html
<div id="SampleElement">
</div>
```

You'll end up with such an HTML markup:

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

TemplatedWidget automatically locates the template for your class and applies it to the HTML element.

## TemplatedWidget ID Generation

If you watch carefully, in our template we specified ID for descendant elements as `~_MyToolbar` and `~_MyTable`.

But when this template is applied to the HTML element, resulting markup contained ID's of **MySamples_MyComplexWidget1_MyToolbar** and `MySamples_MyComplexWidget1_MyTable` instead.

TemplatedWidget replaces prefixes like `~_` with the widget's `UniqueName` and underscore ("_") (`this.idPrefix` contains the combined prefix).

Using this strategy, even if the same widget template is used in a page for more than one HTML element, their ID's won't conflict with each other as they will have unique ID's.

## TemplatedWidget.ByID Method

As TemplateWidget appends a unique name to them, the ID attributes in a widget template can't be used to access elements after widget creation.

Widget's unique name and an underscore should be prepended to the original ID attribute in the template to find an element:

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

TemplatedWidget's ByID method can be used instead:

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

## TemplatedWidget.GetTemplateName Method

In the last sample `MyComplexWidget` located its template automatically.

TemplatedWidget makes use of a convention to find its template (convention based programming). It inserts `Template_` prefix before the class name and searches for a `SCRIPT` element with this ID attribute (`Template_MyComplexWidget`) and uses its HTML content as a template.

If we wanted to use another ID like following:

```html
<script id="TheMyComplexWidgetTemplate" type="text/html">
	...
</script>
```

An error like this would be seen in the browser console:

```text
Can't locate template for widget 'MyComplexWidget' with name 'Template_MyComplexWidget'!
```

We might fix our template ID or ask the widget to use our custom ID:

```cs
public class MyComplexWidget
{
	protected override string GetTemplateName()
    {
    	return "TheMyComplexWidgetTemplate";
    }
}
```

## TemplatedWidget.GetTemplate Method

`GetTemplate` method might be overriden to provide a template from another source or specify it manually:

```cs
public class MyCompleWidget
{
	protected override string GetTemplate()
    {
    	return $('#TheMyComplexWidgetTemplate').GetHtml();
    }
}
```

## Q.GetTemplate Method and Server Side Templates

Default implementation for `TemplatedWidget.GetTemplate` method calls `GetTemplateName` and searches for a `SCRIPT` element with that ID.

If no such SCRIPT element is found, `Q.GetTemplate` is called with the same ID.

An error is thrown if neither returns a result.

`Q.GetTemplate` method provides access to templates defined on the server side. These templates are compiled from files with `.template.cshtml` extension in `~/Views/Template` or `~/Modules` folders or their subfolders.

For example, we could create a template for MyComplexWidget in a server side file like `~/Views/Template/SomeFolder/MyComplexWidget.template.cshtml` with the following content:

```html
<div id="~_MyToolbar">
</div>
<table id="~_MyTable">
	<thead><tr><th>Name</th><th>Surname</th>...</tr></thead>
    <tbody>...</tbody>
</table>
```

Template file name and extension is important while its folder is simply ignored.

By using this strategy there would be no need to insert widget templates into the page markup.

Also, as such server side templates are loaded on the first use (*lazy loading*) and cached in the browser and the server, page markup doesn't get polluted with templates for widgets that we might never use in a specific page. Thus, server side templates are favored over inline SCRIPT templates.


# TemplatedDialog Class

TemplatedWidget's subclass TemplatedDialog makes use of jQuery UI Dialog to create in-page modal dialogs.

Unlike other widget types TemplatedDialog creates its own HTML element, which it will be atteched to.


```
.
.
.
.
.
.
.
.
.
.
.
```





  [1]: WidgetHierarchy.png
