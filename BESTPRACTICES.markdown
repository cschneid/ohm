Ohm Best Practices
==================

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
