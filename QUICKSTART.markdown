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

The package manager for your environment probably has a package for Redis.
Redis 2+ is strongly preferred, but Ohm works with any version from 1.3.10 and
beyond.

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

Custom Validations
------------------

The real power of validations in Ohm is that you can write domain specific
validations.  For example, instead of using a format validation for email, you
can create an assert_email validation.

    def assert_email(attr)
      assert read_local(attr) =~ /^.*@.*$/, [attr, :not_email]
    end

    def validate
      assert_email :email
    end

Blog Example 
=============

This is a quick runthrough of how to architect the database layer of a
hypothetical blog.  We will focus entirely on the Ohm layer, and gloss over any
web related items.

Check the examples/blog directory for a full source file of this example.

Starting off
------------

Before we do anything, we will need to import Ohm.

    require 'rubygems'
    require 'ohm'

And connect to Redis using default options

    Ohm.connect

The Post 
---------

The core of a blog is the list of posts.

    class Post < Ohm::Model
      attribute :title
      attribute :slug
      attribute :body
      index :slug

      # When we save, calculate the slug if we need to, then continue on with
      # Ohm's normal save behavior
      def save
        calculate_slug if slug.nil?
        super
      end

      def validate
        assert_present :title
        assert_present :slug
        assert_present :body

        assert_unique :slug
      end

      def calculate_slug
        slug = title.gsub(/\W+/, '-'). # Remove non-word chars
                     gsub(/^-+/,'').   # Remove a leading dash
                     gsub(/-+$/,'').   # Remove a trailing dash
                     downcase
      end
    end


The Author 
-----------
    class Author < Ohm::Model
      collection :posts, Post

      attribute :name
      attribute :email

      def validate
        assert_present :name
        assert_present :email
      end
    end

    class Post < Ohm::Model
      reference :author, Author
    end

The Comments 
-------------
    class Comment < Ohm::Model
      reference :post, Post

      attribute :body

      def validate
        assert_present :body
      end
    end

    class Post < Ohm::Model
      collection :comments, Comment
    end


Finding Posts 
--------------

When you start off, there won't be any posts.

    Post.all # => []
    Post.size # => 0

Later, you can find your posts

    Post.find(:slug => "ohm-rocks")

Or search by author.  This is a bit tricky, since .find() always returns a
collection, even if it's empty.  So you need to .all to convert to an array,
then [0] to get the first element of that array.

    Author.find(:name => "Chris").all[0].posts


Creating Posts 
---------------

    chris = Author.find(:name => "Chris").all[0].posts
    Post.create({:title => "Ohm Rocks Part 2", :body => "yep, it really does rock", :author => chris})


Creating Comments 
------------------

    p = Post[10]
    p.comments << Comment.create(:body => "Very insightful, Ohm does indeed rock")

Looping Over Comments
---------------------

This is simple, and just like normal Ruby. 

    p = Post[10]
    p.comments.each do |comment|
      puts comment.body
    end

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


read\_local write\_local
----------------------
What are these really for?

How to best describe them

