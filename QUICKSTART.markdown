Ohm Quickstart Guide
====================

What is Ohm?
------------

Ohm is a library that makes serializing your Ruby objects to Redis simple and
fast.

What is Redis?
--------------

Redis is an ultra-fast Key/Value store with lots of features. Redis supports
several data types, including Hashes, Lists, Sets, Counters.


How is Key/Value different than Relational DBs?
-----------------------------------------------



Is Ohm right for me?
--------------------

Ohm is only one of many ways of representing your data.  It does not attempt to
cover all situations, or uses.

You should use Ohm if:

* You need super-fast data access


You should not use Ohm if:

* Real-time Ad-hoc queries are important.  If you don't know how you will query
  your data ahead of time, Ohm can be slower than a traditional relational
  database.

Getting Ohm 
------------

Ohm is available as a rubygem.  Install it as with any other Rubygem.  Add
`sudo` if it is approprate for your environment.

    gem install ohm

Installing Redis 
-----------------

The package manager for your environment probably has a package for Redis.  Be
sure it installs at least version 1.3.10.  Ohm is compatible with Redis 2 as
well.

    port install redis
    brew install redis
    apt-get install redis

Redis also compliles very easily, follow the
[instructions](http://code.google.com/p/redis/wiki/QuickStart) to get started.

Importing Ohm Into Your Project 
--------------------------------

Just like any other library, import Ohm via rubygems.

    require 'rubygems'
    require 'ohm'

Connecting to Redis
-------------------

To setup a global connection for Ohm, call the Ohm.connect function with the
correct options for your environment.  See the extras/ directory for an example
redis.conf file that will work with the command below.

    Ohm.connect(:host => "127.0.0.1", :port => 6379)

Setting Up Your First Model 
----------------------------

Models are just classes that inherit from Ohm::Model, and define attributes, indexes.

    class User < Ohm::Model
      attribute :email
      attribute :encrypted_password

      index :email

      collection :posts
    end

Attribute types 
----------------

Attribute is just a string value stored in Redis.

    class Book < Ohm::Model
      attribute :title  
    end

Counter is an atomically updating integer that you can shift up or down.

    class Count < Ohm::Model
      counter :things 
    end

    c = Count.create
    c.incr(:things)
    c.decr(:things)
    c.things # => 0

Collection is a foreign key type.  This is equivalent to has_many in Active
Record.  

Reference is the other side of a collection.  It is equivalent to belongs_to in
Active Record.

    class Book < Ohm::Model
      collection :authors, Author
    end

    class Author < Ohm::Model
      reference :book, Book
    end

    b = Book.create
    a = Author.create
    a2 = Author.create
    b.authors << a
    b.authors << a2

    b.authors # => [a, a2]
    a.book  # => b

Indexes
-------

Because Key/Value databases don't support searching directly, Ohm maintains a
set of indexes that you tell it, which enables fast searching.

Creating / Retrieving / Deleting / Finding / Sorting 
-----------------------------------------------------

*New Unsaved*

    u = User.new(:email => "foo@example.com")

*Validate*

    u.valid?  # => true / false

*Save*

    u.save # => true / false.  Won't save if validation fails

*Create* and *Save*

    u = User.create(:email => "foo@example.com")

*Find*

    u = User[10] # => Find record #10

*Delete*
  
    u.delete

*Find*
    
    u.find(:email => "foo@example.com")

    u.find(:status => "enabled").except(:name => "Chris")

*Sort*

    User.all.sort # => Sorted by id
    User.all.sort_by(:email)

*Combined Find & Sort*

    User.find(:status => "enabled").sort_by(:email)

Validations
-----------

Ohm makes it easy to verify that the data in your database is valid.

    class User < Ohm::Model
      attribute :name
      attribute :email
      index :name

      def validate
        assert_present :name
        assert_present :email

        assert_unique :name # => Requires the index :name above

        assert_format :email, /^.*@.*$/ # => Make sure the email address has a @ sign in it.
      end
    end

    u = User.create()
    u.valid? # => false
    u.errors # => [[:name, :not_present], [:email, :not_present], [:email, :format]]

Ohm doesn't try to render human readable errors.  That's a job for the view
layer of whatever application you are writing.

    error_messages = u.errors.present do |e|
      e.on [:name, :not_present], "Name must be present"
      e.on [:email, :not_present], "You must supply an email address"
      e.on [:email, :format], "Your email needs an @ sign"
    end

    error_messages
    # => ["Name must be present", 
    #     "You must supply an email address", 
    #     "Your email needs an @ sign"]

Now you have nicely formatted error messages defined in your view, ready to be displayed.

Blog Example 
=============

The post 
---------

The author 
-----------

The comments 
-------------

Finding posts 
--------------

Creating Posts 
---------------

Creating Comments 
------------------


RANDOM STUFF
============

Redis Data Security
-------------------

Redis has a tuneable tradeoff between data security and speed.  By default,
Redis keeps all updates in memory for a few seconds, before writing them
to disk.  This gives Redis massive speed updates, but does have the
tradeoff of potentially losing some data in a system crash.  By changing
a config file, you can have Redis write every change to the disk in
real-time, preventing potential data loss.


Behind the Scenes
=================

What keys is your data in?
--------------------------

