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
