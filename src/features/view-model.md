## View Models

### Introduction 

Probably the most essential feature of any templating engine is a means to easily transform a data structure into HTML using, well, a template. There seem to be almost endless ways to define the view logic: From rather limited approaches like merely replacing placeholders, to defining a complete domain specific language. 

While details of course differ, interestingly enough, most engines seem to agree on embedding the view logic into the template, mixing HTML and custom markup or language constructs into a single file. For rendering, the data is mapped from simple DTOs, xml, json, or - most frequently - associative arrays.

These approaches have a tendency to become a mess quickly, being hard to test and to reason about. Wouldn't it be much nicer to have standard HTML files, ready to preview at any time and otherwise controlling the rendering process from outside the template?

#### Using RDFa

Instead of mixing view logic with markup by embedding instructions into the HTML for a clean separation of concerns, we'd want the logic part to be outside the markup. Yet, for a templating engine to understand how and where to operate, some sort of reference information needs to be provided nevertheless. The best way to do this is by enriching the HTML with structuring metadata, giving the HTML a semantic structure and providing meaning beyond the markup required for a browser to render it:

```html
<section property="author">
    <h3 property="name">John Doe</h3>
    <a property="email" typeoOf="business" href="mailto:john@example.org">E-Mail me</a>
</section>
```

The above HTML fragment has been annotated with [RDFa](https://www.w3.org/TR/html-rdfa/) attributes, a W3C standard to enrich markup with semantic information. In this case, only the `property` attribute has been used though. When following the tree structure, one can now understand that the text content of the "h3" element contains the `name` of an `author` and that his `business` `email` address is "john@example.org".

Instead of reinventing the wheel and coming up with a proprietary set of attributes, Templado makes use of these standard attributes to map data from a View Model into the HTML. This is done by calling the relevant method from or by accessing a public property of the given view model. Depending on the returned type, Templado will decide what to do next. 

#### How rendering works

The view model renderer operates on the internal DOM representation of the template: Starting from the root element - also called `DocumentElement` -, Templado iterates over all its children and also follows the paths down each respective subtree recursively. Whenever one of the RDFa attributes [property](#property), [typeof](#typeof) or [vocab](#vocab) is encountered, the method associated with it will get called on the current view model. If the returned value for a `property` triggered call happens to be an object which is not an iterator or generator, it's considered a view model and thus replaces the view model for the current tree position and all their nested elements - until potentially overwritten by yet another view model. This effectively allows for walking down the document tree.

This probably is best explained with an example. Let's assume this very basic HTML fragment is our template:

```html
<article property="article">
    <h1>Article Headline</h1>    
    <section property="intro">
        <h2>Headlaine without property attribute</h2>
        <p property="content">This is the default text</p>
    </section>
</body>
```

The above example contains a html fragment with a nested structure. While there could be any number of elements in between, basically only the `property` attributes - or rather their respective values - are relevant in this example for the model nesting. Walking down the tree, the nesting is `article > intro > content`. That means, we initially need a view model we can call `article()` on. As we're having a nested structure and want to walk down the tree, we need `article()` to return an object. On the returned object, Templado will call `intro()`. That's because the next element with a property attribute found while walking down the tree is the `section` element and the `property`'s value is "intro". Given we want to continue walking the tree, this method should also return an object. We again turn it into a context view model. The next element with a `property` attribute is the `p` element. Templado thus calls `content()` on the current view model, which is the one returned from `intro()` before. Assuming we want to set the text of the `p` element, we want that method to return a string.

Expressed in PHP code, this could look like this:   

```php
class MyRootViewModel {
    public function article(): Article {
        return new Article();
    }
}

class Article {
    public function section(): Section {
        return new Section();
    }
}

class Section {
    public function p(): string {
        return 'View Model text for p element here';
    }
}
```

Given the required methods do not conflict, and we do not have any need for actual view logic other than the nesting, we could - for this very simple example at least - put all this into a single view model class:

```php
class MyRootViewModel {
    public function article(): self {
        return $this;
    }
    public function section(): self {
        return $this;
    }
    public function p(): string {
        return 'View Model text for p element here';
    }    
}
```

So in other words: By default, the document structure and the placement of `property` attributes mainly define the required nesting of view models. Of course there are exceptions to every rule and Templado provides two means to avoid or rather break out of the nesting: By using a [prefix](#prefix) or by specifying a [resource](#resource). Please refer to the respective paragraph for more details on those.

### Implementing a View Model

To be used as a Templado View Model the implementing class generally is not required to fulfill any Templado specific interface, as you might have guessed given the above examples. While that may seem odd from an object-oriented programming perspective, the methods or properties that need to be provided highly depend on the HTML markup and the used RDFa annotations. From Templado's perspective, there is nothing specific enough to require it in terms of an interface. All we need is it to be an object.

NOTE: Many templating engines accept nested associative arrays as models, what other programming languages call a dictionary. As PHP unfortunately does not differentiate between lists and dictionaries, it is close to impossible to reliably tell whether the key and its value are supposed to be relevant or should be rather ignored. It was thus a deliberate design decision to **not** support associative arrays as models. Templado treats everything that is iterable as a list to iterate over, ignoring potential keys and their values.

If all you have is an associative array, you could build a simple object wrapper around it using PHP's magic method `__get`:

```php
class ArrayToObjectWrapper {
    public function __construct(private array $data) {}

    public function __get(string $key): mixed {
        if (!array_key_exists($key, $this->data)) {
            return null;
        }
        
        $value = $this->data[$key];
        
        if (!is_array($value)) {
            return $value;
        }
        
        if (array_is_list($value)) {
            return $value;
        }
        
        return new self($value);
    }
}
```

WARNING: The above wrapper example uses [`array_is_list`](https://www.php.net/manual/en/function.array-is-list.php) to determine whether the array provided is a dictionary or a list. PHP considers an array a list if its keys consist of consecutive numbers from 0 to count($array)-1. This may not apply to your data structure and thus could be unreliable.  

### Understood RDFa Attributes

While the RDFa standard contains a lot of attributes, Temoplado only looks for a small subset when performing its rendering work. This section describes which attributes are understood, and what the expected behaviour of the view model is for each.

#### vocab

RDFa in HTML is an open standard to semantically enhance markup. To make sense of a set of RDFa nodes, they commonly get semantically grouped into so-called vocabularies. Think of a class or even multiple classes grouping various properties into a meaningful _something_. This _something_ would be your vocabulary in RDFa's terminology.

Of course not all vocabularies are yours and thus most are not relevant for view model rendering. So in case your template contains RDFa attributes from other vocabularies, Templado needs to get told which ones to operate on.

NOTE: If you do not include third party vocabularies in your templates, you do not need to set the `vocab` attribute nor implement support into your view model.

Whenever a `vocab` attribute is encountered while walking down the DOM tree, Templado checks whether the current view model implements a `vocab()` method. If not, the attribute is ignored. Otherwise, Templado will call the method, passing in the value of the `vocab` attribute. The return value is expected to be a string that Templado then can compare against the specified vocab - as in, the value of the vocab attribute. If they match, Templado considers the view model to be responsible for this vocabulary and will try to render it.

Again, this is probably best understood with an example. A simple HTML fragment, that has a `vocab` attribute set could look like this:

```html
<p vocab="https://example.com#book">Hello world</p>
```

To specify a view model that is considered responsible for this vocabulary, various options exist. All the following examples lead to the same result.

```php
class MyRootModel {
    public function vocab(string $requested): string {         
        return $requested;
    }
}
```

```php
class MyRootModel {
    public function vocab(): string {
        return "https://example.com#book";
    }
}
```

```php
class MyRootModel {
    public string $vocab = "https://example.com#book";
}
```

If you want to __not__ have the view model be applied, you have to return a none-matching string. So if the requested vocabulary would for instance be "https://schema.org", the last two examples above would already qualify as not being responsible.

The first example simply returns the input string and thus would always be considered responsible. The implementation could just as well be removed as it's technically identical to _not_ having a `vocab()` method at all - but some prefer to make it explicit.

#### property

The `property` annotation is the attribute used to tell Templado to apply or change content - as can be seen to some extent in the examples above already. In general, the value of this attribute is interpreted as the name of the method to be called on the current view model or the name of the public property to be read from it.
 
If the value of this attribute contains a `:` though, Templado splits the string at the location of the double colon, interpreting the first part as a [prefix](#prefix) and the remainder as the method to call or property to access. Given a prefix is set, the current view model context is ignored and the model to operate on is resolved using the found prefix. For this to succeed, the [prefix](#prefix) has to have been bound to an object beforehand. Please read the description of [prefix](#prefix) to learn more about this feature.

When the `property` annotation resolves to a method call, the current textual value of the context element is passed along by Templado.
This allows for simple placeholders to replaced or existing textual content to be extended without having to duplicate it into the view model.

NOTE: While the textual content of an element may be technically composed out of various (even nested) markup elements, only their respective text is passed along. E.g. `<p>hello <b>world</b> out there!</p>` would translate to `hello world out there!`, effectively stripping any markup contained.

##### Supported Types 

Templado expects the type of the accessed property or the value returned by a method to be either a string, an array or object - where some object types do have special meaning as listed below. For backwards compatibility, a boolean type is also supported. Returning any other type will cause Templado to throw an exception.

###### String

When a `string` is returned, Templado will replace the current element's content with its value. Be aware that this effectively also removes any other children the current element might have had:

```html
<p property="templado">Hello <b>world</b> out there!</p>
```

```php
class StringViewModel {
    public string $templado = 'Templado';    
}
```

```html
<p property="templado">Templado</p>
```

###### Signal::ignore (or true)

Sometimes, Templado needs to be told to skip over an element even though a property attribute had been set and just continue as if none would have been. This can be achieved by returning a `Signal::ignore()`. For backwards compatibility, a boolean `true` can also be used. 

Ignoring an element will not change the current view model context.

###### Signal::remove (or false)

To have the current element and thus all its children be removed from the document, a `Signal::remove()` can be returned. For backwards compatibility, a boolean `false` can also be used.

Removing an element will not change the current view model context.

###### Iterables (e.g. Arrays or Iterator)

Iterable types are Templado's equivalent of "foreach". The current element will be cloned and the content adjusted for as many times as there are items returned while iterating.

NOTE: To preview how for example a list of items would look like, the template document may contain the same element with the same `property` value multiple times. Templado will extract and clone the first one to copy it as many times as needed while iterating, removing all additional elements with the same property value in the current context.

Each iterative call to the view model must either yield a string or new view model object. Direct nesting of iterables or signaling is not supported. 

###### View Model Objects

*** DOCUMENTATION PENDING ***




##### Operating on HTML Attributes

Apart from setting the text content of an element, Templado supports changing the value of or entirely removing existing attributes. Templado will *not* add new attributes.

NOTE: To not break Templado's flow of operation, the supported RDFa attributes are protected and can **not** be changed or removed using a view model. If you prefer to have RDFa attributes removed at the end of processing, this can be achieved by using a [Serializer](serializing.md) or [Transformation](transformation.md). 


###### Supported Types for Attribute Rendering

**String**

**Signal::ignore (or true)**

**Signal::remove (or false)**



#### resource

*** DOCUMENTATION PENDING ***

#### prefix

*** DOCUMENTATION PENDING ***

#### typeof

*** DOCUMENTATION PENDING ***





