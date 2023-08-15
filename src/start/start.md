## Getting Started

As with any templating engine, the main goal is to pair a template with a data source. Templado uses View Models and HTML templates to accomplish this. So the first thing we need to do is introduce them to each other.

### 1. Loading the Template

For Templado to operate, we need a Template to work with. From Templado's point of view, every Template is a `Document`. This can either be a complete HTML page - or just a fragment or snippet. 

A Templado `Document` can be instantiated either from a string or by supplying an already created instance of PHP's `DOMDocument`. For our first steps, we'll be using a String:

```php
$document = Templado\Engine\Document::fromString('
    <!DOCTYPE html>
    <html lang="en">
        <head>
            <title>Hello World, Templado!</title>
        </head>
        <body>
            <h1 property="headline">My First Template</h1>
        </body>
    </html>
');
```

NOTE: While HTML can be represented using either the HTML or the XML serialization format, LibXML 2.x - the underlying engine that powers most of PHP's DOM support - does only support the HTML serialization format up to version 4. For that reason, and to avoid other quirks, Templado uses the XML mode when parsing a string. If you do not want this, feel free to create a `DOMDocument` by other means and pass that to Templado using the alternative factory method `Templado\Engine\Document::fromDomDocument`.

### 2. Applying a View Model

Once a `Document` instance is available, a View Model can be applied. Templado relies on [RDFa](https://www.w3.org/TR/html-rdfa/) attributes embedded into the HTML to determine which methods to call on the current view model. More details on this, supported attributes and their respective meaning for Templado as well as more complex constructs can be found in the [View Models](../features/viewmodel.md) section. 

The above basic HTML example contains a single RDFa attribute `property`. Templado uses this attribute, or rather the value of it, as the method name to call on the current view model and to determine what to do.

The application of a very basic view Model to change text of the `<h1>` element could look like this:

```php
$document->applyViewModel(new class {
    public function headline(): string {
        return 'Hello world!';
    }
});
```

The `property`'s value is "headline", thus Templado is going to call the method named "headline" on the view model. Given the returned type is a `string`, the element's text content will be changed.

### 3. Serializing back to HTML

Given the above sample model, Templado will replace the original text ('My First Template') with 'Hello world!'. To verify our success, we can have Templado serialize the Document back to HTML for us. We'll do so by using Templado's HTML Serializer. More on the serializer support can be found in the [Serializer](../features/serializing.md) section. The HTML Serializer used here will ensure we produce sane HTML 5 output, despite the fact we use the XML mode internally:

```php
print $document->asString(
    new Templado\Engine\HTMLSerializer()
);
```

The above should produce this output:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Hello World, Templado!</title>
    </head>
    <body>
        <h1 property="headline">Hello world!</h1>
    </body>
</html>
```

**Congratulations, you just rendered your first HTML page using Templado!**
