# EnumerateIt

💎 Ruby Enumerations with magic powers! 🎩

[![Build Status](https://travis-ci.org/lucascaton/enumerate_it.svg?branch=master)](https://travis-ci.org/lucascaton/enumerate_it)
[![Gem Version](https://badge.fury.io/rb/enumerate_it.svg)](https://rubygems.org/gems/enumerate_it)
[![Code Climate](https://codeclimate.com/github/lucascaton/enumerate_it/badges/gpa.svg)](https://codeclimate.com/github/lucascaton/enumerate_it)
[![Changelog](https://img.shields.io/badge/changelog--brightgreen.svg?style=flat)](https://github.com/lucascaton/enumerate_it/releases)

**EnumerateIt** helps you to declare and use enumerations in a very simple and flexible way.

### Why would I want a gem if Rails already has native enumerations support?

Firstly, **EnumerateIt** works amazingly well along with **Rails** but it is not required!
It means you can add it to any **Ruby** project! 😀

Secondly, Rails' enumerations have a problem:
[`ActiveRecord::Enum`](http://edgeapi.rubyonrails.org/classes/ActiveRecord/Enum.html) uses
`integers` instead of `strings`, which means that if you change your enumeration list oder in your
model, your database will no longer be consistent.
Database storage pricing is not a problem nowadays, so it's recommended to use `strings` columns.

- [Installation](#installation)
- [Using with Rails](#using-with-rails)
- [Creating enumerations](#creating-enumerations)
  - [Sorting enumerations](#sorting-enumerations)
- [Using enumerations](#using-enumerations)
- [I18n](#i18n)
- [Using enumerations to handle a legacy database](#using-enumerations-to-handle-a-legacy-database)
- [FAQ](#faq)
- [Changelog](#changelog)

## Installation

```bash
gem install enumerate_it
```

## Using with Rails

Add the gem to your Gemfile:

```ruby
gem 'enumerate_it'
```

Run the install generator:

```bash
rails generate enumerate_it:install
```

There is also a Rails Generator which generates enumerations and their locale files:

```bash
rails generate enumerate_it:enum --help
```

## Creating enumerations

Enumerations are created as classes and you should put them inside `app/enumerations` folder.

You can pass an array of symbols, so that the respective value for each symbol will be the
stringified version of the symbol itself:

```ruby
class RelationshipStatus < EnumerateIt::Base
  associate_values(
    :single,
    :married,
    :divorced
  )
end
```

This will create some nice stuff:

* Each enumeration's value will turn into a constant:

  ```ruby
  RelationshipStatus::SINGLE
  # => 'single'

  RelationshipStatus::MARRIED
  #=> 'married'
  ```

* You can retrieve a list with all the enumeration codes:

  ```ruby
  RelationshipStatus.list
  #=> ['divorced', 'married', 'single']
  ```

* You can retrieve a JSON with all the enumeration codes:

  ```ruby
  RelationshipStatus.to_json
  #=> "[{\"value\":\"divorced\",\"label\":\"Divorced\"},{\"value\":\"married\", ...
  ```

* You can get an array of options, ready to use with the 'select',
  'select_tag', etc family of Rails helpers.

  ```ruby
  RelationshipStatus.to_a
  #=> [['Divorced', 'divorced'], ['Married', 'married'], ['Single', 'single']]
  ```

* You can retrieve a list with values for a group of enumeration constants.

  ```ruby
  RelationshipStatus.values_for %w(MARRIED SINGLE)
  #=> ['married', 'single']
  ```

* You can retrieve the value for a specific enumeration constant:

  ```ruby
  RelationshipStatus.value_for('MARRIED')
  #=> 'married'
  ```

* You can retrieve the symbol used to declare a specific enumeration value:

  ```ruby
  RelationshipStatus.key_for(RelationshipStatus::MARRIED)
  #=> :married
  ```

* You can iterate over the list of the enumeration's values:

  ```ruby
  RelationshipStatus.each_value { |value| ... }
  ```

* You can iterate over the list of the enumeration's translations:

  ```ruby
  RelationshipStatus.each_translation { |translation| ... }
  ```

* You can also retrieve all the translations of the enumeration:

  ```ruby
  RelationshipStatus.translations
  ```

* You can ask for the enumeration's length:

  ```ruby
  RelationshipStatus.length
  #=> 3
  ```

### Sorting enumerations

When calling methods like `to_a`, `to_json` and `list`, the returned values will be sorted using
the translation for each one of the enumeration values. If you want to overwrite the default sort
mode, you can use the `sort_by` class method.

```ruby
class RelationshipStatus < EnumerateIt::Base
  associate_values married: 1, single: 2

  sort_by :translation
end
```

The `sort_by` methods accept one of the following values:

| Value          | Behavior                                                                                     |
| :------------- | :------------------------------------------------------------------------------------------- |
| `:none`        | The default behavior, will return values in order that was passed to `associate_values` call |
| `:translation` | will sort the returned values based on translations                                          |
| `:name`        | Will sort the returned values based on the name of each enumeration option                   |

## Using enumerations

The cool part is that you can use these enumerations with any class, be it an `ActiveRecord`
instance or not.

```ruby
class Person
  extend EnumerateIt
  attr_accessor :relationship_status

  has_enumeration_for :relationship_status
end
```

> **Note:** **EnumerateIt** will try to load an enumeration class based on the camelized attribute
> name. If you have a different name, you can specify it by using the `with` option:
>
> `has_enumeration_for :relationship_status, with: RelationshipStatus`

This will create:

* A "humanized" version of the hash's key to humanize the attribute's value:

  ```ruby
  p = Person.new
  p.relationship_status = RelationshipStatus::DIVORCED
  p.relationship_status_humanize
  #=> 'Divorced'
  ```

* A translation for your options, if you include a locale to represent it
  (see more in the [#i18n](I18n section):

  ```ruby
  p = Person.new
  p.relationship_status = RelationshipStatus::DIVORCED
  p.relationship_status_humanize
  #=> 'Divorciado'
  ```

* The associated enumerations, which can be retrieved with the `enumerations` class method:

  ```ruby
  Person.enumerations
  #=> { relationship_status: RelationshipStatus }
  ```

* A helper method for each enumeration option, if you pass the `create_helpers` option as `true`:

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status, with: RelationshipStatus, create_helpers: true
  end

  p = Person.new
  p.relationship_status = RelationshipStatus::MARRIED

  p.married?
  #=> true

  p.divorced?
  #=> false
  ```

  It's also possible to "namespace" the created helper methods, passing a hash to the `create_helpers`
  option. This can be useful when two or more of the enumerations used share the same constants:

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status,
      with: RelationshipStatus, create_helpers: { prefix: true }
  end

  p = Person.new
  p.relationship_status = RelationshipStatus::MARRIED

  p.relationship_status_married?
  #=> true

  p.relationship_status_divorced?
  #=> false
  ```

  You can define polymorphic behavior for the enumeration values, so you can define a class for each
  of them:

  ```ruby
  class RelationshipStatus < EnumerateIt::Base
    associate_values :married, :single

    class Married
      def saturday_night
        'At home with the kids'
      end
    end

    class Single
      def saturday_night
        'Party Hard!'
      end
    end
  end

  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status,
      with: RelationshipStatus, create_helpers: { polymorphic: true }
  end

  p = Person.new
  p.relationship_status = RelationshipStatus::MARRIED
  p.relationship_status_object.saturday_night
  #=> 'At home with the kids'

  p.relationship_status = RelationshipStatus::SINGLE
  p.relationship_status_object.saturday_night
  #=> 'Party Hard!'
  ```

  You can also change the suffix `_object`, using the `suffix` option:

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status,
      with: RelationshipStatus, create_helpers: { polymorphic: { suffix: '_mode' } }
  end

  p.relationship_status_mode.saturday_night
  ```

  The `create_helpers` also creates some mutator helper methods, that can be used to change the
  attribute's value.

  ```ruby
  p = Person.new
  p.married!

  p.married?
  #=> true
  ```

* A scope method for each enumeration option if you pass the `create_scopes` option as `true`:

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status, with: RelationshipStatus, create_scopes: true
  end

  Person.married.to_sql
  #=> SELECT "users".* FROM "users" WHERE "users"."relationship_status" = "married"
  ```

  The `:create_scopes` also accepts :prefix option.

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status,
      with: RelationshipStatus, create_scopes: { prefix: true }
  end

  Person.relationship_status_married.to_sql
  ```

* An inclusion validation (if your class can manage validations and responds to
  `validates_inclusion_of`):

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status, with: RelationshipStatus
  end

  p = Person.new(relationship_status: 'invalid')
  p.valid?
  #=> false
  p.errors[:relationship_status]
  #=> 'is not included in the list'
  ```

* An presence validation (if your class can manage validations and responds to
  `validates_presence_of` and you pass the `required` options as `true`):

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status, required: true
  end

  p = Person.new relationship_status: nil
  p.valid?
  #=> false
  p.errors[:relationship_status]
  #=> "can't be blank"
  ```

  If you pass the `skip_validation` option as `true`, it will not create any validations:

  ```ruby
  class Person < ActiveRecord::Base
    has_enumeration_for :relationship_status, with: RelationshipStatus, skip_validation: true
  end

  p = Person.new(relationship_status: 'invalid')
  p.valid?
  #=> true
  ```

Remember that you can add validations to any kind of class and not only to those derived from
`ActiveRecord::Base`.

### Why to define enumerations outside the class that uses them?

* It's clearer.
* You can add behaviour to the enumeration class.
* You can reuse the enumeration inside other classes.

## I18n

I18n lookup is provided on both `_humanized` and `Enumeration#to_a` methods, given the hash key is
a Symbol. The I18n strings are located on `enumerations.<enumeration_name>.<key>`:

```yaml
# Your locale file
pt-BR:
  enumerations:
    relationship_status:
      married: Casado
```

```ruby
class RelationshipStatus < EnumerateIt::Base
  associate_values(
    :married,
    :single
  )
end

p = Person.new
p.relationship_status = RelationshipStatus::MARRIED
p.relationship_status_humanize
#=> 'Casado'

p.relationship_status = RelationshipStatus::SINGLE
p.relationship_status_humanize # Non-existent key
#=> 'Single'
```

You can also translate specific values:

```ruby
status = RelationshipStatus::MARRIED
RelationshipStatus.t(status)
#=> 'Casado'
```

## Using enumerations to handle a legacy database

**EnumerateIt** can help you build a Rails application around a legacy database which was filled
with those small and unchangeable tables used to create foreign key constraints everywhere, like the
following example:

```sql
Table "public.relationship_status"

  Column     |     Type      | Modifiers
-------------+---------------+-----------
 code        | character(1)  | not null
 description | character(11) |

Indexes:
  "relationship_status_pkey" PRIMARY KEY, btree (code)

SELECT * FROM relationship_status;

code |  description
---- +--------------
1    | Single
2    | Married
3    | Divorced
```

You might also have something like a `users` table with a `relationship_status` column and a foreign
key pointing to the `relationship_status` table.

While this is a good thing from the database normalization perspective, managing these values in
tests is very hard. Doing database joins just to get the description of some value is absurd.
And, more than this, referencing them in the code using magic numbers was terrible and meaningless:
What does it mean when we say that someone or something is `2`?

In this case, you can pass a hash:

```ruby
class RelationshipStatus < EnumerateIt::Base
  associate_values(
    single:   1,
    married:  2,
    divorced: 3
  )
end
```

```ruby
RelationshipStatus::MARRIED
#=> 2
```

You can also sort it by its value: `sort_by :value`.

## FAQ

#### What versions of Ruby and Rails are supported?

Please check out [travis config file](https://github.com/lucascaton/enumerate_it/blob/master/.travis.yml).

#### Can I set a value to always be at the end of a sorted list?

Yes, please see [issue #60](https://github.com/lucascaton/enumerate_it/issues/60).

## Changelog

Changes are maintained under [Releases](https://github.com/lucascaton/enumerate_it/releases).

## Note on Patches/Pull Requests

* Fork the project.
* Make your feature addition or bug fix.
* Add tests for it. This is important so we don't break it in a future version unintentionally.
* [Optional] Run the tests agaist a specific Gemfile: `$ bundle exec appraisal rails_5.0 rake spec`.
* Run the tests agaist all supported versions: `$ bundle exec rake` (or `$ bundle exec wwtd`)
* Commit, but please do not mess with `Rakefile`, version, or history.
* Send a Pull Request. Bonus points for topic branches.

## Copyright

Copyright (c) 2010-2017 Cássio Marques and Lucas Caton. See `LICENSE` file for details.
