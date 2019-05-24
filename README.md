# Templado Documentation
A pragmatic approach to templating for PHP 7.2+

[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/templado/engine/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/templado/engine/?branch=master)
[![Code Coverage](https://scrutinizer-ci.com/g/templado/engine/badges/coverage.png?b=master)](https://scrutinizer-ci.com/g/templado/engine/?branch=master)
[![Build Status](https://scrutinizer-ci.com/g/templado/engine/badges/build.png?b=master)](https://scrutinizer-ci.com/g/templado/engine/build-status/master)

[![Build Status](https://travis-ci.org/templado/engine.svg?branch=master)](https://travis-ci.org/templado/engine)

[![SensioLabsInsight](https://insight.sensiolabs.com/projects/f5b424bf-2a00-4df8-8070-db7f0d03e4e9/mini.png)](https://insight.sensiolabs.com/projects/f5b424bf-2a00-4df8-8070-db7f0d03e4e9)


### Motivation

Most of today's templating engines mix code for the required rendering logic with HTML markup in one file and require
the developers to learn their respective language.

Templado follows a different approach on templating: Being in part inspired by [Tempan](https://github.com/watoki/tempan),
Templado relies solely on plain HTML markup. The limited amount of display logic required is contained with the engine
and triggered by the view model when it's applied to the Page.

### Always ready to preview

As a Templado template is plain HTML, previewing is as easy as opening the HTML file with a browser - example data can
and should be included as the engine will clean it up based on the view model upon rendering.

### No markup duplication
 
Templado features asset support, mapping a list of assets based on their ID into a given HTML Page. To automate this
process [Templado CLI](https://github.com/theseer/templado-cli) can be used. Combined with a File watcher in your IDE,
you can have an always up-to-date set of HTML pages without ever writing a block twice.

### Form handling included

To make form handling even more easy, Templado comes with explicit HTML Form support. Based on supplied Input data,
Templado will repopulate the HTML form and even include your CSRF protection code.

### Custom transformations and Filters

Templado allows for custom transformations, like adding a class to every ```a``` tag and string based replacements upon
serialization.

## Installation

**Note that PHP 7.2+ is required** 

The preferred method for installing Templado is to simply add it to your project using Composer. 

If you are manually creating or editing the composer.jon file:

```
"require" : {
    "templado/engine": "^3.0"
}
```

Or from the command line:

`# composer require templado/engine:^3.0`

For basic usage, this is all that is necessary. Templado makes use of View Model objects in order to make the magic 
happen. In addition, you have the option of using either JSON, or XML (*More on this later*). In this case you will also 
need to install the mappers, which are in a separate repository.  


```
"require" : {
    "templado/engine": "^3.0"
    "templado/mappers": "^1.0"
}
```

or from the command line:

`# composer require templado/mappers:^1.0`


## Getting Started

As with any templating engine, the goal is to pair a template with a data source. Templado uses View Models and XHTML 
templates to accomplish this. So the first thing we need to do is introduce them to each other.

### Associate A Template With A View Model

Here is an example of code to pair a data model with a template:

```
try {
    $page = Templado\Engine\Templado::loadHtmlFile(
        new Templado\Engine\FileName(__DIR__ . '/html/viewmodel.xhtml')
    );
    $page->applyViewModel(new ViewModel());

    echo $page->asString() . "\n";

} catch (Templado\Engine\TempladoException $e) {
    foreach($e->getErrorList() as $error) {
        echo (string)$error;
    }
}
```

Let's quickly parse this code. 

First, we instantiate a Templado\Engine\Html object. To accomplish this we use the static method 
Templado\Engine\Templado::loadHtmlFile. *Notice that we create a new Templado FileName object to pass into the method.* 

Once loaded, we call the "applyViewModel" method, and pass in our View Model. 

This Object does not have to be called "ViewModel". The name is arbitrary. It is called ViewModel here for clarity.

Now that the pairing is complete, you can simply call the "asString" method to output the rendered file. 

Finally, we wrap it in a try/catch, and that's it. With this you are ready to begin templating.

## Basic Usage

One of the great features of Templado is that it relies entirely on HTML markup, with no need for you to learn any 
new language or syntax. Templado has the ability to modify all aspects of your html template. Let's start with basic 
markup elements.

### Elements and Attributes

Let's say that you have a header tag in your template. 

`<h1>Some Title</h1>`

Templado looks for an attribute in each element specifically named "property". If it finds one, it looks for a method in 
your View Model who's name matches the attribute's value.

`<h1 property="headline">Some Title</h1>`

In this example, Templado would access the View Model for this template, and look for a method named "headline".

```
class ViewModel {
    public function headline() {
        return "The Actual Title";
    }
}
```

Continuing with this example, the method is returning a string, and so Templado would use the returned string as the 
text value for the header:

`<h1 property="headline">The Actual Title</h1>`

It's that simple, and this will work for any HTML element. However it gets much better. Instead of returning a string, 
your View Model method can return an Object representing the HTML element. This Object can have methods that represent 
any of the attributes of your HTML element ...

`<h1 property="headline" title="A Title" class="a-class">Some Title</h1>`

In this example our header now has two attributes - "title" and "class". So we can create a class that looks like this:

```
class Headline {
    public function title() {
        return "Awsome Title";
    }
    
    public function class() {
        return "cool-class";
    }
    
    public function asString() {
        reuturn "The Actual Title";
    }
}
```

And then return a new instance of our Headline object from the View Model method.

```
class ViewModel {
    public function headline() {
        return new Headline();
    }
}
```

When an Object is returned Templado will look for methods in the Object that match the names of the attributes of the 
element in the template. 

**Note that Templado looks for a method specifically called "asString" for the text value of the element.**

Now our header will render like this:

`<h1 property="headline" title="Awsome Title" class="cool-class">The Actual Title</h1>`

In addition, if for any reason you have an attribute in your template, but you want to remove it for this data model, 
you can simply return false from the method. So if our Headline object returned false for the class method, our rendered 
header would then look like this:

`<h1 property="headline" title="Awsome Title">The Actual Title</h1>`

Pretty cool, but there is one more feature to discuss about elements and attributes.

Templado captures the values of elements and attributes from the template. And if a method in your Object takes a 
parameter, then the value is passed in. 

So let's say your Object looks like this now: 

```
class Headline {
    public function title() {
        return "Awsome Title";
    }
    
    public function class($original) {
        return $original . " cool-class";
    }
    
    public function asString() {
        reuturn "The Actual Title";
    }
}
```

You see that we have added a parameter called $original (*the parameter name is up to you*), and we then concatenate the 
the class from the template with our new class, and return that. 

So now our header would render like this:

`<h1 property="headline" title="Awsome Title" class="a-class cool-class">The Actual Title</h1>`

Again this will work with all elements and attributes. 

The fact is that you could stop reading right here, and with only these few features accomplish many of your templating 
goals! You can create an Object to represent any HTML element and all of its attributes. This is cool, but keep reading. 
There is much more. 

### Dynamic Lists

One special case/feature of Templado is that a View Model method can return an array. This allows for multiples of an 
element to be rendered consecutively with different data for each. The most obvious usage for this is with lists. 

Let's say that you need an unordered list of items displayed. When you are creating your template, you have no idea 
of the number of items that will need to be shown. This is not an issue. 

In your template you can create the unordered list with a single list item element, and give it a "property" attribute:

```
<ul>
    <li property="items">Item 1</li>
</ul> 
```

Then in your View Model, the "items" method can return an array of list items:

```
class ViewModel {
    public function items() {
        return [
            "Item 1", 
            "Item 2",
            "Item 3" 
        ];
    }
}
```

In this case the rendered elements would like this:

```
<ul>
    <li property="items">Item 1</li>
    <li property="items">Item 2</li>
    <li property="items">Item 3</li>
</ul> 
```

Notice that in this example we returned an array of strings, so Templado used the strings as the text for each list item.
Just as with our previous element examples, you can also return an array of Objects. These Objects will represent each 
list item element ... just as before.

In the template:

```
<ul>
    <li property="items" class="odd">Item 1</li>
</ul> 
```

In the View Model:

```
class ViewModel {
    public function items() {
        return [
            new Item("odd", "Item 1"), 
            new Item("even", "Item 2"),
            new Item("odd", "Item 3")
        ];
    }
}
```

And of course you would need an Item class:

```
class Item {
    /**
     * @var string
     */
    private $class;
    
    /**
     * @var string
     */
    private $value;
    
    public function __construct($class, $value) {
        $this->$class = $class;
        $this->value = $value;
    }
    
    public function class() {
        return $this->class;
    }
    
    public function asString() {
        return $this->value;
    }
}
```

And now the rendered elements would like this:

```
<ul>
    <li property="items" class="odd">Item 1</li>
    <li property="items" class="even">Item 2</li>
    <li property="items" class="odd">Item 3</li>
</ul> 
```

One of the nicest features about Templado is that your template files can be rendered in a browser without the 
associated data. You can see how your template looks without concern over what data will be passed to it. 

Given this, there may be times that you want your template to look more fleshed out. For example, even though you don't 
know exactly how many items will be displayed, you are sure that in most cases it will be several. Let's say you expect 
an average of about 5 items. The final page will look very different than your raw template if the template only has one 
item as a placeholder.

This is not a problem. With Templado, you can add as many elements as you deem necessary in the template. 

So your template can look like this:

```
<ul>
    <li property="items" class="odd">Item 1</li>
    <li property="items" class="even">Item 2</li>
    <li property="items" class="odd">Item 3</li>
    <li property="items" class="even">Item 4</li>
    <li property="items" class="odd">Item 5</li>
</ul> 
```

But your rendered elements will still look like this if we use the View Model from before:

```
<ul>
    <li property="items" class="odd">Item 1</li>
    <li property="items" class="even">Item 2</li>
    <li property="items" class="odd">Item 3</li>
</ul> 
```

If your View Model method returns ten items, then the rendered list will have ten items. 

The bottom line is that you can design your templates to look realistic, and still know that they will render dynamically. 
This is a nice advantage over other templating engines.


### Nesting

It should be relatively obvious at this point, but to be explicit, just as HTML can have many levels of nesting, so can 
your View Models.

Take this template as an example:

```
<div property="user">
    <p>Name: <span property="name">Original Name</span></p>
    <div>
        <span>EMail:</span>
        <ul>
            <li property="emailLinks">
                <a property="email" href="mailto:original@domain.tld" class="current">original@domain.tld</a>
            </li>
        </ul>
    </div>
</div>
```
Here the top level element has a property called "user". Within the scope of the "user", we have two properties - "name" 
and "emailLinks". And within the emailLinks scope we have a property called "email". So from a property point of view we 
have 3 levels of nesting. 

So our View Model will follow this schema as follows:

```
class ViewModel {
    public function user() {
        return new User();
    }
}
```

And the User:

```
class User {

    public function name() {
        return 'Willi Wichtig';
    }

    public function emailLinks() {
        return [
            new EMailLink(
                new Email('willi@wichtig.de')
            ),
            new EMailLink(
                new Email('second@secondis.de')
            )
        ];
    }

}
```

When the scope changes in the template, it follows in the Models. So now Templado is looking at the User Model for the 
"name" and "emailLinks" methods. Notice that the "emailLinks" method is returning an array of EmailLink objects.
So each EmailLink is rendered in the list, and the Templado looks to each of the EmailLink objects to resolve the 
email property. So our EmailLink object looks like this:

```
class EMailLink {

    /** @var  Email */
    private $email;

    /**
     * @param Email $email
     */
    public function __construct(Email $email) {
        $this->email = $email;
    }

    public function email() {
        return $this->email;
    }
}
```

And finally, our Email object corresponds to an anchor element in our template. So Templado is looking for methods which 
match the attributes of the anchor:

```
class Email {

    /** 
     * @var string 
     */
    private $addr;

    public function __construct(string $addr) {
        $this->addr = $addr;
    }

    public function asString() {
        return $this->addr;
    }

    public function href() {
        return 'mailto:' . $this->asString();
    }
    
    public function class() {
        return false;
    }
}
``` 

Notice that we returned false for the class property, which removes it from the rendered output. We could have, as another 
example, set a flag in our Email object to signify the "current" or "preferred" email, and then output that 
dynamically. The options become almost limitless for you as you create more complex templates. The nesting and dynamic 
complexity is up to you and the requirements of your project.

### Special Attributes - Prefix and Resource

There are two special attribute features in Templado. The first is called "prefix", and is simply a way to alias 
(or namespace) your properties. Let's say that you have a property called "user" that you want to reference multiple 
times. You can use the "prefix" attribute in place of the "property" attribute to alias your element. Then prefix 
properties using the colon notation:

```
<div prefix="u user">
    <p>Name: <span property="u:name">Original Name</span></p>
    <div>
        <span>EMail:</span>
        <ul>
            <li property="u:emailLinks">
                <a property="email" href="mailto:original@domain.tld" class="current">original@domain.tld</a>
            </li>
        </ul>
    </div>
</div>
```

The second special attribute is called "resource". Let's say that you have a User element nested within another element 
in your template.

```
<div property="accountDetails">
    <div property="user">
        <p>Name: <span property="name">Original Name</span></p>
        <div>
            <span>EMail:</span>
            <ul>
                <li property="emailLinks">
                    <a property="email" href="mailto:original@domain.tld" class="current">original@domain.tld</a>
                </li>
            </ul>
        </div>
    </div>
</div>
```

But perhaps you also want to display the User element within another section of the page. In this case, rather than 
duplicating the method code in each section, you can use the "resource" attribute. This attribute works in 
the same way as the "property" attribute with one important difference. In stead of looking within the current scope, 
Templado goes back to your top level View Model to find the method. So we change the above template code, and replace 
the "property" attribute for the User element to be "resource".

```
<div property="accountDetails">
    <div resource="user">
        <p>Name: <span property="name">Original Name</span></p>
        <div>
            <span>EMail:</span>
            <ul>
                <li property="emailLinks">
                    <a property="email" href="mailto:original@domain.tld" class="current">original@domain.tld</a>
                </li>
            </ul>
        </div>
    </div>
</div>
```

And now Templado will look for the user method in ViewModel, rather than AccountDetails. 



## Examples

Usage examples can be found in the [example project](https://github.com/templado/examples)