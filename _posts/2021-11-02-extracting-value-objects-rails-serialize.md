---
title: Write clean Object-Oriented code by extracting value objects in Rails
description: As Ruby on Rails developers, we tend to add too much code in the same ActiveRecord model. Let's exctract some of it with the help of the serialize method
date: 2021-11-02
minutes_read: 6
tags: [ruby, rails]
---

As Ruby on Rails developers, we tend to add too much code in the same `ActiveRecord model`. Very often, this is because we fail to identify new objects. This article aims to present a concrete method to keep our `ActiveRecord` models sizes under control thanks to the [serialize method](https://github.com/rails/rails/blob/main/activerecord/lib/active_record/attribute_methods/serialization.rb).

## Identifying value objects in Rails

Let's take a concrete example. Imagine that we are working on an invoicing system, each invoice being composed of multiple line items. Our Invoice model would look like this:

```rb
class Invoice < ApplicationRecord
  has_many :line_items, dependent: :destroy
end
```

In our example, a `LineItem` will have a `quantity` stored as an integer and a `unit_price` stored as a decimal. The `LineItem` model would look like this:

```rb
class LineItem < ApplicationRecord
  belongs_to :invoice

  validates :quantity, numericality: { only_integer: true, greater_than: 0 }
  validates :unit_price, numericality: { greater_than: 0 }

  def total_price
    (unit_price * quantity).round(2)
  end
end
```

Let's now imagine a new requirement: we have to handle VAT rates of 0%, 10%, and 20% on line items, 0% VAT rate being for tourist taxes. The most straightforward implementation would probably be to add all of the VAT rates in the `LineItem` model as it already exists. We would add a tax_code string field in our `line_items` table, and our `LineItem` model would start growing:

```rb
class LineItem < ApplicationRecord
  # previous associations and validations

  TAX_CODES = { vat_0: 0.0, vat_10: 0.1, vat_20: 0.2 }

  validates :tax_code, inclusion: { in: TAX_CODES.keys.map(&:to_s) }

  def tax_rate
    TAX_CODES[tax_code.to_sym]
  end

  def tourist_tax?
    tax_code.to_sym == :vat_0
  end

  # We now have to take the tax_rate into account
  def total_price
    (unit_price * quantity * (1 + tax_rate)).round(2)
  end
end
```

This is a very simple example of course but failing to identify value objects creates two issues:

- Our `LineItem` model is doing too much. Failing to identify new objects will make the model longer and longer, and it will harm readability sooner or later.
- Tax-related information is now entangled inside of our LineItem model. If we want to reuse tax codes elsewhere in our application, we are stuck.

Tax codes represent something important in our application and we should give them a proper place to live.

## Extracting value objects in Rails

We are first going to give this object a new home and a new name. With a few minor changes, we have extracted all the methods in the `TaxCode` object:

```rb
# app/models/tax_code.rb

class TaxCode
  TAX_CODES = { vat_0: 0.0, vat_10: 0.1, vat_20: 0.2 }

  attr_reader :key

  def initialize(key)
    @key = key.to_sym
  end

  def tax_rate
    TAX_CODES[key]
  end

  def tourist_tax?
    key == :vat_0
  end
end
```

Our `LineItem` model is now much smaller and focused on its main responsibility as we removed all the tax code related code:

```rb
class LineItem < ApplicationRecord
  # previous associations and validations

  validates :tax_code, inclusion: { in: TaxCode::TAX_CODES.keys.map(&:to_s) }

  def total_price
    (unit_price * quantity * (1 + tax_rate)).round(2)
  end

  def tax_rate
    TaxCode.new(tax_code).tax_rate
  end
end
```

There is still one issue to solve: The `LineItem#tax_code` does not return a `TaxCode` object. Let's demonstrate this issue in the rails console:

```shell
line_item = LineItem.create(quantity: 1, unit_price: 100, tax_code: :vat_20)
line_item.tax_code
=> "vat_20"
```

We want to get a `TaxCode` object when calling the `LineItem#tax_code` method. Luckily, Rails provides a way to do precisely that, thanks to the serialize method. Let's instruct the LineItem model that its tax_code method should return a TaxCode object:

```rb
class LineItem < ApplicationController
  serialize :tax_code, TaxCode

  # all the previous code...
end
```

For the `.serialize` method to work, `TaxCode` should implement both the `.load` and `.dump` class methods. As mentioned in [the documentation](https://github.com/rails/rails/blob/main/activerecord/lib/active_record/attribute_methods/serialization.rb#L51), `.dump` will be called to serialize an object and should return the serialized value to be stored in the database. On the other hand, `.load` will be called to reverse the process and deserialize from  the database. In our case, that means converting a string into a `TaxCode` object.

Let's add those two class methods to the `TaxCode` class:

```rb
# app/models/tax_code.rb

class TaxCode
  def self.load(key)
    # We don't want to build TaxCodes with a nil key
    new(key) if key
  end

  def self.dump(tax_code)
    case tax_code
    when self then tax_code.key
    else tax_code
    end
  end

  # all the previous code...
end
```

Great! Now the `LineItem#tax_code` method should return a `TaxCode` object! Let's update the validations and the `LineItem#total_price` method accordingly:

```rb
class LineItem
  # all the previous code
  validates :tax_code, inclusion: { in: TaxCode.all }

  def total_price
    (unit_price * quantity * (1 + tax_code.tax_rate)).round(2)
  end
end
```

Let's add the `.all` method on `TaxCode` and the `#==` method to be able to compare two tax codes as it is necessary for the inclusion validation to be able to compare them:

```rb
# app/models/tax_code.rb

class TaxCode
  # all the previous code...

  def self.all
    TAX_CODES.keys.map { |key| new(key) }
  end

  # Tax codes are equal if they have the same key
  def ==(other)
    if other.is_a?(self.class)
      key == other.key
    end
  end
end
```

Let's restart our rails console and try again:

```shell
line_item = LineItem.create(quantity: 1, unit_price: 100, tax_code: :vat_20)
line_item.tax_code
=> <TaxCode:0x00007fd0c6316728 @key=:vat_20>
```

The final implementation
That was a lot of work! Let's look at the final implementation of the `LineItem` model:

```rb
class LineItem < ApplicationRecord
  belongs_to :invoice

  validates :quantity, numericality: { only_integer: true, greater_than: 0 }
  validates :unit_price, numericality: { greater_than: 0 }
  validates :tax_code, inclusion: { in: TaxCode.all }

  serialize :tax_code, TaxCode

  def total_price
    (unit_price * quantity * (1 + tax_code.tax_rate)).round(2)
  end
end
```

As we can see, our model is very small and only focused on line items related calculations. All taxes related code has been extracted to the TaxCode class.

We can notice the same beneficial effects in the `TaxCode` class. All tax-related code is isolated in a small class with a single responsibility:

```rb
# app/models/tax_code.rb

class TaxCode
  TAX_CODES = { vat_0: 0.0, vat_10: 0.1, vat_20: 0.2 }

  def self.load(key)
    new(key) if key
  end

  def self.dump(tax_code)
    case tax_code
    when self then tax_code.key
    else tax_code
    end
  end

  def self.all
    TAX_CODES.keys.map { |key| new(key) }
  end

  attr_reader :key

  def initialize(key)
    @key = key.to_sym
  end

  def tax_rate
    TAX_CODES[key]
  end

  def tourist_tax?
    key == :vat_0
  end

  def ==(other)
    if other.is_a?(self.class)
      key == other.key
    end
  end
end
```

Our `TaxCode` model is now completely extracted and can be reused anywhere inside our application!

## Serialization in the wild

Admittedly, our value object example is a bit simple. Let's explore a real-world example. `ActionText` is a part of the Ruby on Rails framework and enables us to add rich text to our applications with just one line of code. To add rich text to an `Article` model for example, we would simply add the following line to our model:

```rb
class Article < ApplicationRecord
  has_rich_text :content
end
```

Under the hood, the [`.has_rich_text`](https://github.com/rails/rails/blob/main/actiontext/lib/action_text/attribute.rb#L33) method defines a `has_one` association with an object of class `ActionText::RichText`. The `ActionText::RichText` object holds the rich text inside the database in a body field that gets [serialized](https://github.com/rails/rails/blob/main/actiontext/app/models/action_text/rich_text.rb#L11) in an `ActionText::Content` object.

```rb
module ActionText
  class RichText < Record
    serialize :body, ActionText::Content

    # ...
  end
end
```

The [`ActionText::Content`](https://github.com/rails/rails/blob/main/actiontext/lib/action_text/content.rb) object does much more than our simple `TaxCode` example of course; it defines many more methods! The `.load` and `.dump` methods are defined in the [`ActionText::Serialization` concern](https://github.com/rails/rails/blob/main/actiontext/lib/action_text/serialization.rb) that is included in [`ActionText::Content`](https://github.com/rails/rails/blob/main/actiontext/lib/action_text/content.rb#L5).
