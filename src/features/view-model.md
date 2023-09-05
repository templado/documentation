## View Models

### Introduction 

Probably the most essential feature of any templating engine is a means to easily transform a data structure into HTML using, well, a template. There seem to be almost endless ways to define the view logic: From rather limited approaches like merely replacing placeholders, to defining a complete domain specific language. 

While details of course differ, interestingly enough, most engines seem to agree on embedding the view logic into the template, mixing HTML and custom markup or language constructs into a single file. For rendering, the data is mapped from simple DTOs, xml, json, or - most frequently - associative arrays.

These approaches have a tendency to become a mess quickly, being hard to test and to reason about. Wouldn't it be much nicer to have standard HTML files, ready to preview at any time and otherwise controlling the rendering process from outside the template?

#### RDFa

Instead of mixing view logic with markup by embedding instructions into the HTML for a clean separation of concerns, we'd want the logic part to be outside the markup. Yet, for a templating engine to understand how and where to operate, some sort of reference information needs to be provided nevertheless. The best way to do this is by enriching the HTML with structuring metadata, giving the HTML a semantic structure and providing meaning beyond the markup required for a browser to render it:

```html
<section property="author">
    <h3 property="name">John Doe</h3>
    <a property="email" typeoOf="business" href="mailto:john@example.org">E-Mail me</a>
</section>
```

The above HTML fragment has been annotated with [RDFa](https://www.w3.org/TR/html-rdfa/) attributes. In this case, only the `property` attribute has been used though. When following the tree structure, one can now understand that the text content of the "h3" element contains the `name` of an `author` and that his `business` `email` address is "john@example.org".

Instead of reinventing the wheel and coming up with a proprietary set of attributes, Templado makes use of these attributes to map data from a View Model into the HTML. This is done by calling the relevant method from or by accessing a public property of the given view model. Depending on the returned type, Templado will decide what to do next. 

### How Templado's ViewModel Rendering works

The view model renderer operates on the internal DOM representation of the template: Starting from the root element - also called `DocumentElement` -, Templado iterates over all its children and also follows the paths down each respective subtree recursively. Whenever one of the RDFa attributes [vocab](#vocab), [property](#property) or [typeof](#typeof) is encountered, the method associated with it will get called on the current view model context. If the returned value for a `property` triggered call happens to be an object, it's considered a view model and thus replaces the view model for the current tree position and all their nested elements - until potentially overwritten by yet another view model. This effectively allows for walking down the document tree.

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

### Operating on HTML Attributes

Apart from setting the text content of an element, Templado generally supports changing the value of or entirely removing attributes. 

NOTE: To not break Templado's flow of operation, the supported RDFa attributes are excluded and can **not** be changed or removed using a view model.   

### Implementing a View Model

To be used as Templado View Model the implementing class generally is not required to fulfill any templado specific interface, as you might have guessed given the above examples. While that may seem odd from an object-oriented programming perspective, the methods or properties that need to be provided highly depend on the HTML markup and the used RDFa annotations. From Templado's perspective, there is nothing specific enough to require in terms of an interface - except it being an object.

NOTE: Many templating engines accept nested associative arrays as models. As PHP unfortunately so far does not differentiate between lists and dictionaries, it is close to impossible to tell whether the key and its value is supposed to be relevant or should be rather ignored. It was thus a deliberate design decision to **not** support associative arrays as models. Templado will treat everything that is iterable as a list.  

### Supported Return Types for `property`

#### String

#### Signal::ignore (or true)

#### Signal::remove (or false)

#### Iterables (Arrays or Iterator)

#### View Model Objects


### Supported RDFa Attributes

While the RDFa standard contains a lot of attributes, Temoplado only looks for a small subset when performing its rendering work. This section describes which attributes are understood, and what the expected behaviour of the view model is for each.

#### vocab

RDFa in HTML is an open standard to semantically enhance markup. To make sense of a set of RDFa nodes, they commonly get semantically grouped into so-called vocabularies. Think of a class or even multiple classes grouping various properties into a meaningful _something_. This _something_ would be your vocabulary in RDFa's terminology.  

Of course not all vocabularies are yours and thus most are not relevant for view model rendering. So in case your template contains RDFa attributes from other vocabularies, Templado needs to get told which ones to operate on. 

NOTE: If you do not include third party vocabularies in your templates, you do not need to set the `vocab` attribute nor implement support into your view model.

Whenever a `vocab` attribute is encountered while walking down the DOM tree, Templado checks whether the current view model implements a `vocab()` method. If not, the attribute is ignored. Otherwise, Templado will call the method, passing in the value of the `vocab` attribute. The return value is expected to be a string that Templado then can compare against the specified vocab - as in, the value of the vocab attribute. If they match, Templado considers the view model to be responsible for this vocabulary and will try to render it.

A simple HTML fragment, that has a `vocab` attribute set could look like this:

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

If you want to __not__ have the view model be applied, you have to return a none-matching string. So the last two examples above would already qualify as not responsible, if the requested vocabulary would for instance be "https://schema.org". 

The first example simply returns the input string and thus would be considered responsible. Given it always returns the input, the implementation could just as well be removed as it's technically identical to _not_ having a `vocab()` method at all.


#### property

#### resource

#### prefix

#### typeof

