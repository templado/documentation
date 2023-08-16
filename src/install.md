## Installation

### Runtime Requirements

Templado merely requires an up-to-date PHP Version with XML and DOM support.
So this boils down to the following:  

- PHP >= 8.2.0
- Extensions
    - dom
    - libxml

NOTE: Please note that when you want to contribute to the development of Templado or if you just want to run the tests or some of the tools used during development of Templado like infection or psalm, additional extensions are required. As this is not a runtime requirement, those are not listed here.

### Install with Composer

Templado is designed to be installed as a library using [Composer](https://getcomposer.org), the defacto standard to install and manage runtime dependencies for PHP.

The easiest way to add Templado to your project is from the CLI:

```shell
$ composer require templado/engine:^5.0
```

If you prefer to manually create or edit the `composer.json` file, please add the following fragment to it.

```json
"require" : {
    "templado/engine": "^5.0"
}
```

For Templado, and its dependencies, to be actually installed after manually editing, you'd have to explicitly run `composer install`.


### Manual Installation

If you do not want to or cannot use Composer for some reason, a manual installation is of course also possible. Keep in mind that you are fully responsible to manage a potential upgrade, dependency resoling and autoloading. If you still insist on a manual installation, feel free to clone the source repository, and switch to whatever is the latest 5.x release tag. For instance:

```bash
git clone git@github.com:templado/engine.git && cd engine && git switch 5.x.y
```

For the CSSSelector to work, you need to additionally install `theseer\css2xpath`:

```bash
git clone git@github.com:theseer/css2xpath.git
```

NOTE: As you opted for not using Composer, no autoloader has been generated yet. Please update your autoloader accordingly, so the newly installed classes and interfaces can be found.   

WARNING: While of course technically possible, using a manual installation is *not* supported. If you run into any issues with Templado, please ensure the problem is reproducible using a composer based installation before reporting it.

