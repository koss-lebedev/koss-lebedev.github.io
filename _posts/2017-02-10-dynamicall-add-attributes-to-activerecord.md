---
title: How to dynamically add attributes to your ActiveRecord models
tags:
- ruby
---

![](<../images/dynamic-attributes.png>)

Sometimes we need to build an application that has domain models that we don’t know all the attributes of. A good example of such application is a system for tracking business contacts. In the center of it is a Contact model that has attributes like name, email, phone number, etc. But can we know beforehand all the attributes our Contact model will need to have?  If we want to create the application for a wide audience, it can be difficult to predict. Solution? Allow users to add more attributes to Contact model in the runtime! We’ll do exactly that in this tutorial.

You can find the complete source of the demo application [here.](https://github.com/koss-lebedev/active_dynamic_demo) I won’t be covering the basic setup for this application (which, indeed, is very basic), and instead, focus on interesting parts.

Configuring model
-----------------

First thing first, let’s install [active_dynamic](https://github.com/koss-lebedev/active_dynamic) gem. I wrote this gem while working on the application similar to the one we'll build in this tutorial, and the gem will do most of the heavy-lifting for us. Start by adding it to your Gemfile and running:

```shell
rails generate active_dynamic
rake db:migrate
```

This will copy migration and configuration files to your project and run the migration.

Next, we need to include active_dynamic into our Contact model so that the gem knows that Contact can have dynamically added attributes:

```ruby
class Contact < ActiveRecord::Base
  has_dynamic_attributes

  # ... other code
end
```

Now we need to tell active_dynamic where to find the information about attributes it has to add to Contact model. To do that, we create a provider class that has to comply with two rules:

- it needs to accept a model class as the only constructor argument;
- it needs to have a `call` method that returns an array of attribute definitions;

Attribute definition is just a helper class that allows us to specify a display name of an attribute that will be dynamically added to our model, and, optionally, its data type, a system name, presence validation, and a default value. Here’s a simple provider class that defines `age` and `description` attributes:

```ruby
class ContactAttributeProvider

  def initialize(model)
    @model = model
  end

  def call
    [
      ActiveDynamic::AttributeDefinition.new('age', datatype: ActiveDynamic::DataType::Integer, default_value: 18),
      ActiveDynamic::AttributeDefinition.new('description')
    ]
  end

end
```

To make them truly dynamic, we can load attribute definitions from an external storage — a database, YAML file, XML config, or something else. In the demo application, I used a `contact_attributes` table to store and manipulate the list of attribute definitions, so my ContactAttributeProvider looks like this:

```ruby
class ContactAttributeProvider

  def initialize(model)
    @model = model
  end
  def call
    ContactAttribute.all.map do |attr|
      ActiveDynamic::AttributeDefinition.new(
        attr.name, datatype: attr.datatype,
        required: attr.required?)
    end
  end
end
```

Finally, we need to tell active_dynamic to use this provider class. In the configuration file `config/initializers/active_dynamic.rb` add the line:

```ruby
config.provider_class = CustomAttributeResolver
```
Now every time an instance of Contact is created, it will have whatever attributes you listed in your attribute provider class. So now we can use it like this:

```ruby
contact = Contact.new

# first_name is a regular attribute mapped to
# a column in contacts table
contact.first_name = 'John'

# age attribute was added dynamically
contact.age = 27

contact.save

# we can also do this
second_contact = Contact.new(first_name: 'Jane', age: 25)
```

Configuring controller and view
-------------------------------

Now that our model is ready, let’s create a view to edit it.

In the demo application, I used [simple_form](https://github.com/plataformatec/simple_form) gem to generate HTML forms, but the same approach can be applied using standard  Rails form builder. Here is the helper method that maps an array of dynamic attributes to an array of input fields:

```ruby
def dynamic_attribute_inputs(form, model)
  inputs = model.dynamic_attributes.map do |attr|
    options = {
        label: attr.display_name,
        as: datatype_mapping(attr.datatype)
    }
    form.input attr.name, options
  end

  safe_join inputs
end

def datatype_mapping(type)
  case type
    when ActiveDynamic::DataType::Text then :text
    when ActiveDynamic::DataType::Integer then :integer
    else :string
  end
end
```

It iterates over dynamic attributes of a model and uses `datatype` property that we set earlier to create different input types for different types of attributes. So it will render a text area element for `text` data type, numeric input for `integer` data type, and a simple text input for any other data type. We then can use this helper method in our view like this:

```erb
<%= simple_form_for(contact) do |f| %>
  ... code omitted
  <%= f.input :first_name %>
  <%= f.input :last_name %>
  <%= dynamic_attribute_inputs(f, contact) %>

  <%= f.button :submit %>
<% end %>
```

The last missing piece is in the controller. We need to white-list our dynamic attributes for mass assignment. We can do that like this:

```ruby
def contact_params
  params.require(:contact).permit(
    *Contact.new.dynamic_attributes.map(&:name), :first_name)
end
```

Now when we create or update a Contact record, we can use contact_params hash that will include all of our dynamic attributes.

That’s it! Now we have a complete solution that allows us to add new attributes for Contact model as well as set their values without making changes in the source code.

* * *

If you have any questions, hit me up on [twitter](https://twitter.com/koss_lebedev) or [add an issue](https://github.com/koss-lebedev/active_dynamic/issues) on GitHub. Thanks for reading!
