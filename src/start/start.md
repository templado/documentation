## Getting Started

As with any templating engine, the main goal is to pair a template with a data source. Templado uses View Models and HTML templates to accomplish this. So the first thing we need to do is introduce them to each other.

### Load the Template Markup

For Templado to operate, an instance of `Document` needs to be created. This can be done either from a string - which will then be parsed internally into a `DOMDocument` or by supplying an already created instance of PHP's `DOMDocument`.

```php
$document = Templado\Engine\Document::fromString('
    <!DOCTYPE html>
    <html lang="en" >
        <head>
            <title>Hello World, Templado!</title>
        </head>
        <body>
            <h1 property="headline">My First Template</h1>
        </body>
    </html>
'); 
```

WARNING: As Templado uses PHP's DOMDocument, the markup must be a valid XML string. HTML 5 can be written using the HTML or XML serialization format. LibXML, the underlying engine that powers PHP's DOM support, does not support the HTML serialization format of HTML 5. Templado does, nevertheless, produce valid HTML 5 using its own HTMLSerializer.  

### Applying a View Model

Once the `Document` is available, a View Model can be applied. The above basic HTML example contains a single `property` attribute, which Templado uses as the method name to fetch an alternative Text body.

A basic view Model could look and be applied like this:

```php
$document->applyViewModel(new class {
    public function headline(): string {
        return 'Hello world!';
    }
});

print $document->asString(
    new Templado\Engine\HTMLSerializer()
);

```

