## Migration from Templado 4.x

In general, migrating from Templado 4.x to Templado 5.0 should be pretty straight forward. The general concepts as well as APIs are mostly kept identical or got enhanced in a backwards compatible way.

The main changes are:

- Requires PHP 8.2+
- `HTML` and `Snippet` got logically merged into a `Document`, representing both types
- A new `HTMLSerializer` has been introduced to generate HTML 5 output
- ViewModels now can have
    - public properties to satisfy the rendering needs
    - return a signal instance rather than relying on bool types and null
- Form data handling now supports nested structures as well as fields referenced by id
- A `Selector` can now be used in more cases to specify what areas of a document to apply the change to
- Optionally `vocab` aware
- Enhanced trace output in case of errors
- Snapshot support

The following paragraphs give more in-depth details on these changes.

### Updated and Changed Requirements

In contrast to Templado 4, which was designed to work with PHP 7+, Templado 5 requires PHP 8.2 and later. Additionally, as Templado 5 ships with an enhanced custom HTML 5 Serializer implementation, a new requirement to `ext/xmlwriter` has been added.

### Loading Facade `Templado` removed

In Templado 4.x and earlier, a facade named `Templado` existed providing static factory methods to create `HTML` instances from files or strings. This extra facade was deemed superfluous and got removed.

It's functionality got, in part, replaced by `Document::fromString`. Templado 5 no longer performs any file loading operations itself, so the class `FileName` got removed.

This means the following calls would need to get changed from

```php
$html1 = Templado::loadHTMLFile(new FileName(...));

$html2 = Templado::parseHtmlString(...);
```

to

```php
$html1 = Document::fromString(file_get_contents(...));

$html2 = Document::fromString(...);
```

This should be a simple search-and-replace refactoring.

### Dealing with potential parse errors

As Templado 5 does no longer provide a loading facade, the respective exceptions are also gone. If you want to catch any issues with parsing when using the `Document::fromString` method you can now explicitly catch `Templado\Engine\ParsingException` rather than the old rather generic `TempladoException`. To learn the details on what actually went wrong, you can ask the exception instance object for the `LibXMLError` instances using `Templado\Engine\ParsingException::errors()`.


### Everything is a `Document` now

In Templado 4.x and earlier, every rendering operation could be applied on a document using the `HTML` facade object - optionally created by the aforementioned factory facade. To support merging fragments into documents, the concept of so-called snippets existed, represented by multiple types of `Snippet` classes. In a nutshell, these merely served as a glorified transfer object and implemented different strategies on how the merge should be performed.

This concept turned out to be rather impractical, hard to understand and limiting in practical use, so for Templado 5, the concept of a snippet being different from an HTML document got dropped. Everything is now a `Document`, regardless whether it represents a complete HTML structure or merely a fragment.

Accordingly, throughout a code base using Templado 4, all type declarations need to get changed from `HTML` or `Snippet` to `Document`.

#### Migrating Snippets

To migrate from Templado 4, all explicit calls to load or instantiate a Snippet need to get replaced by calls to `Document` and the best matching factory method. As already mentioned, Templado no longer performs filesystem operations itself. This means, the previously existing `SnippetLoader` used in the following example to create `$snippet3` has been removed:

```php
$snippet1 = new \Templado\Engine\SimpleSnippet('some-id', $dom);

$snippet2 = new \Templado\Engine\TempladoSnippet('some-id', $dom);

$snippet3 = (new SnippetLoader)->load(new FileName('...'), 'optionally-id-here');
```

For merging of documents (or previously snippets), the to-be-merged objects must have an ID set. To ensure the syntactical validity according to W3C rules, a new value object `Id` got introduced and replaces the scalar string type used before.

NOTE: In Templado 4 when using the `SnippetLoader`, providing an ID was optional. If none was given, Templado's loader implementation checked whether an ID was set as an attribute on the root element within the loaded document and extracted it from there. As the loader functionality got removed, this is technically no longer possible. The implicit assignment of the ID from content was confusing at best, and, unfortunately, a common source of errors. 

The above code examples should thus be replaced as follows:

```php
$snippet1 = \Templado\Engine\Document::fromDomDocument($dom, new Id('some-id'));

$snippet2 = \Templado\Engine\Document::fromDomDocument($dom, new Id('some-id'));

$snippet3 = \Templado\Engine\Document::fromString(
    file_get_contents('...'), 
    new Id('no-longer-optional-id-here')
);
```

#### Merging Documents instead of Applying Snippets

As the concept of a technically separate Snippet got dropped, the API to get fragments into a (main) document got renamed from `applySnippets` to `merge`.

This change allows to also merge "fragments" into each other as they also are now just a `Document`.

Furthermore, the API got enhanced to allow for passing one or more `Document` instances directly, without the explicit need for a collection of documents.

```php
$html = Templado::parseHtmlString(...);

$list = new SnippetListCollection();
$list->addSnipet(...);
$list->addSnipet(...);

$html->applySnippets($list);
```

Due to the API change, the above Templado 4 code can be adjusted as follows: 

```php
$document = Document::fromString('...');
$document->merge(
   Document::fromString('...', new Id('...')),
   Document::fromString('...', new Id('...'))
);
```

Of course, alternatively using a collection is still a valid option as well:

```php
$document = Document::fromString('...');

$list = new DocumentCollection(
    Document::fromString('...', new Id('...')),
    Document::fromString('...', new Id('...'))
);

$document->merge($list);
```

#### Snippet Document Namespace changed

Snippets in Templado 4 optionally support having a wrapper element to ensure xml compliance when otherwise multiple root elements would exist:

```xml
<templado:snippet xmlns="http://www.w3.org/1999/xhtml" xmlns:templado="https://templado.io/snippet/1.0">
 <h1>Title</h1>
 <p>text here</p>
</templado:snippet>
```

Templado 5.0 documents provide the same option, but use a different namespace as the id lookup mechanism optionally used in Templado 4 is no longer supported. The root element name in Templado 5 defaults to "document" but is technically irrelevant as only the namespace is currently checked upon merge.

```xml
<templado:document xmlns="http://www.w3.org/1999/xhtml" xmlns:templado="https://templado.io/document/1.0">
 <h1>Title</h1>
 <p>text here</p>
</templado:snippet>
```

---

### TODO

```
Transformation::getSelector => Transformation::selector

HTML::asString() => Document::asString(new HTMLSerializer)
```
