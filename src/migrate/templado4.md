## Migration from Templado 4.x

### Updated and Changed Requirements

In contrast to Templado 4, which was designed to work in PHP 7, Templado 5 requires PHP 8.2 or later. Additionally, as Templado 5 ships with an enhanced custom HTML 5 Serializer implementation, the new requirement of `ext/xmlwriter` has been added.

### Loading Facade `Templado` removed

In Templado 4.x and earlier, a facade named `Templado` existed which provided static factory methods to create `HTML` instances from files or strings. This extra facade was deemed superfluous and got removed.

It's functionality got, in part, replaced by `Document::fromString`. As Templado no longer performs any file loading operations itself, the class `FileName` got removed.

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

#### Migrating Snippets

To migrate from Templado 4, all explicit calls to load or instantiate a Snippet need to get replaced by calls to `Document` and the best matching factory method. As already mentioned, Templado no longer performs filesystem operations itself. This means, the previously existing `SnippetLoader` used in the following example to create `$snippet3` has been removed.

```php
$snippet1 = new \Templado\Engine\SimpleSnippet('some-id', $dom);

$snippet2 = new \Templado\Engine\TempladoSnippet('some-id', $dom);

$snippet3 = (new SnippetLoader)->load(new FileName('...'), 'optionally-id-here');
```

For merging of documents (or previously snippets), the to-be-merged objects must have an ID set. To ensure the syntactical validity according to W3C rules, a new value object `Id` got introduced and replaces the scalar string type used before.

In Templado 4 when using the `SnippetLoader`, providing an ID was optional. If none was given, Templado's loader implementation checked whether an ID was set as an attribute on the root element within the loaded document and extracted it from there. 
As the loader functionality got removed, this is technically no longer possible.

The implicit assignment of the ID from content was confusing at best, and, unfortunately, a common source for errors. 

The above code examples should thus be replaced as follows:

```php
$snippet1 = \Templado\Engine\Document::fromDomDocument($dom, new Id('some-id'));

$snippet2 = \Templado\Engine\Document::fromDomDocument($dom, new Id('some-id'));

$snippet3 = \Templado\Engine\Document::fromString(
    file_get_contents('...'), 
    new Id('no-longer-optional-id-here')
);
```

---

### TODO

```
HTML::applySnippets => Document::merge


SnippetListCollection => DocumentCollection

SnippetListCollection::addSnippet => DocumentCollection::add

Transformation::getSelector => Transformation::selector


Namespace of snippets wrapper document:

snippet : https://templado.io/snippet/1.0 => document : https://templado.io/document/1.0


HTML::asString() => Document::asString(new HTMLSerializer)
```
