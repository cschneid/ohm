Ohm Best Practices
==================

Denormalization with Set & List
-------------------------------

Don't let yourself get stuck with the traditional relational database idea of
normalizing all of your data.  Your app probably has chunks of data that make
sense only as a whole.

Rather than creating a new model for these pieces of data, consider a list, or
set on the parent model, and storing structured data as a string.  This can be 
XML, JSON, YAML, or any other serialization format you'd want.

Don't overuse this approach, but be aware that it it s a great option to
simplify and organize your data model.


Migrations
----------

Testing
-------



Spawning Records
----------------

Ohm does not come with a default way to spawn new records automatically.  We
recommend using the Spawn gem along with the Faker gem to do this quickly and easily.

To install these gems:

    gem install spawner
    gem install faker

Using them is simple:

    class Post < Ohm::Model
      extend Spawner

      attribute :title
      attribute :author
    end

    Post.spawner do |p|
      p.title  ||= Faker::Lorem.words(3).join(" ")
      p.author ||= Faker::Name.name
    end


Then to spawn a new Post, you call:

    # Faked title and author
    Post.spawn

    # Fixed title, faked author
    Post.spawn(:title => "Fixed title, not faker'd")

* [Faker Documentation](http://faker.rubyforge.org/rdoc/)
* [Spawn Documentation](http://github.com/soveran/spawn)

Extensions
----------

Often you'll find yourself doing work that can be automated. For example, a
boolean attribute.

    def loaded?
      read_local(:loaded) == "true"
    end

    def loaded=(val)
      # Normalize to true/false
      val = val ? "true" : "false"
      write_local(:loaded)
    end

Why do that when you can write:

    boolean_attribute :loaded

So abstract your code out, and make that a library.

    module Ohm::BooleanAttributes
      def boolean_attribute(attr)
        attribute attr

        define_method("#{attr.to_s}?") do
          read_local(attr) == "true"
        end

        # overwrites the default attribute version
        define_method("#{attr}=") do |val|
          val = val ? "true" : "false"
          write_lcoal(attr, val)
        end
      end
    end

    class NeedsBooleans < Ohm::Model
      extend Ohm::BooleanAttributes

      boolean_attribute :loaded
    end

Now the logic for boolean attributes is in one reusable place, rather than
repeated and scattered through your code.


