---
layout: post
title: "Enums with Rails & ActiveRecord: an improved way"
originally_published_at: https://sipsandbits.com/2018/04/30/using-database-native-enums-with-rails/
description: ActiveRecord Enums are a really good tool to use when you need that certain model's attribute to have a finite number of possible values.
---

## Overview

ActiveRecord Enums are a really good tool to use when you need that certain model's attribute to have a finite number of possible values.

Its usage is pretty straightforward, see the example below:

```ruby
class Post < ApplicationRecord
  enum status: %i[draft reviewed published]
end
```

In order to persist the **status** value to the database, we need to generate a migration using the following command:

    bundle exec rails g migration AddStatusToPosts

And it should look like this:

```ruby
# db/migrate/20180426164051_add_status_to_posts.rb
class AddStatusToPosts < ActiveRecord::Migration[5.2]
  def change
    add_column :posts, :status, :integer, default: 0
  end
end
```

It's important to mention that, for this case, the column needs to be an **integer**, and it will contain values from *0 to 2*. Setting the default as 0 is a good practice. Those values will mean the following:

- 0 for draft status
- 1 for reviewed status
- 2 for published status

Yes, you noticed it right, **the order we define the values on our **enum** does matter a lot**. And if we decide to add new values, the recommendation is to add them at the end, so we don't mess with data that we may already have in our database.

There is a way for us to override the integer number that will be used to represent a value from our **enum**, you can use the following approach:

```ruby
class Post < ApplicationRecord
  enum status: { draft: 2, reviewed: 1, published: 0 }
end
```

As you can imagine, the mapping has changed to the following:

- 0 for published status
- 1 for reviewed status
- 2 for draft status

Cool, enough with the "mappings", let's talk about the convenience methods that we get when using the **enum**, assuming we have the following **enum declaration**:

```ruby
enum status: %i[draft reviewed published]
```

These are the methods:

```ruby
post = Post.new

post.draft! # => true
post.draft? # => true
post.status # => "draft"

post.reviewed! # => true
post.draft?    # => false
post.status    # => "reviewed"
post.reviewed? # => true
```

Fancy, uh? As you can imagine we get the same methods for our **published** status.

Be aware that **!** methods, change the status and also saves the record, so you need to make sure that your model meets all validations prior using the **bang** methods or they will fail.

Thanks to **?** methods you no longer need to do comparisons to know if your record is in specific status.

The usage of **enums** also adds convenience scopes for us:

```ruby
Post.draft     # => Collection of all Posts in draft status
Post.reviewed  # => Collection of all Posts in reviewed status
Post.published # => Collection of all Posts in published status
```

## Pros

Besides the convenience methods and scopes that I already mentioned to you, the main advantage of using enums is that, out of the box, we get validations for the possible values that a column can have.

So the following example will raise an exception:

```ruby
post = Post.new

post.status = "unknown"
=> ArgumentError ('unknown' is not a valid status)
```

## Cons

### Integer columns are hard to understand without actual context

One of the downsides of using an **integer** column for storing a representation of a string value, like the case of a **status** column, is that we are going to make it harder for people looking directly at the database table's rows to know what number represents which status. In the previous examples, there are 3 possible statuses, but we can many more, there is no limit.

Fortunately, we can make the life easier for the people looking directly to the database.

Instead of using an **integer** column, you can use a **string** one. Your migration should look like this:

```ruby
# db/migrate/20180426164051_add_status_to_posts.rb
class AddStatusToPosts < ActiveRecord::Migration[5.2]
  def change
    add_column :posts, :status, :string
  end
end
```

Next, you need to change your **enum declaration** to the following:

```ruby
class Post < ApplicationRecord
  enum status: {
    draft: "draft",
    reviewed: "reviewed",
    published: "published"
  }
end
```

That's it, know your **posts** table will have a **string** status column and consequently, any person looking at it will know what status the Post has.

### Data Integrity issues

Now that you have already solved the readability issue in your **posts** table. There is still one important issue that we need to resolve. The integrity of the information within your database. Currently, it is totally possible for the people with read/write access to your database to set invalid values on your **status** column. As you may recall, validations are handled within your Rails application context. If we are already using **enum** on Rails, is because we know the values that your **status** column can have is finite.

Let's add the very same constraints we have on our Rails app, to the database. To do so, native databases enums come to our rescue! I will teach you how to write migrations to generate that kind of columns in **MySQL** and **PostgreSQL**.

#### MySQL

The migration should look like this if you are adding the column:

```ruby
# db/migrate/20180426164051_add_status_to_posts.rb
class AddStatusToPosts < ActiveRecord::Migration[5.2]
  def up
    execute <<-SQL
        ALTER TABLE posts ADD status enum('published', 'draft', 'reviewed');
    SQL
  end

  def down
    remove_column :posts, :status
  end
end
```

Interesting, uh? We are using **MySQL's** **enum()** type to declare the possible values that the **status** column should have. You're guessing right, those values should be exactly the same than the ones declared on your Model.

In case your column already exist, your migration should look like this:

```ruby
# db/migrate/20180426164051_change_post_status_column_type.rb
class AddStatusToPosts < ActiveRecord::Migration[5.2]
  def up
    execute <<-SQL
        ALTER TABLE posts MODIFY status enum('published', 'draft', 'reviewed');
    SQL
  end

  def down
    change_column :posts, :status, :string # Previous type
  end
end
```

#### PostgreSQL

The migration should look like this if you are adding the column:

```ruby
# db/migrate/20180426164051_add_status_to_posts.rb
class AddStatusToPosts < ActiveRecord::Migration[5.2]
  def up
    execute <<-SQL
      CREATE TYPE post_statuses AS ENUM ('published', 'draft', 'reviewed');
      ALTER TABLE posts ADD status post_statuses;
    SQL
  end

  def down
    execute <<-SQL
      DROP TYPE post_statuses;
    SQL
    remove_column :posts, :status
  end
end
```

**PostgreSQL** also supports **enums**, but we should define them first. That's why we use the **CREATE_TYPE** command to define our enum, and then we add the **status** column and it uses the previously defined **enum** with **CREATE_TYPE**.

In case your column already exist, your migration should look like this:

```ruby
# db/migrate/20180426164051_change_post_status_column_type.rb
class AddStatusToPosts < ActiveRecord::Migration[5.2]
  def up
    execute <<-SQL
      CREATE TYPE post_statuses AS ENUM ('published', 'draft', 'reviewed');
      ALTER TABLE posts MODIFY status post_statuses;
    SQL
  end

  def down
    execute <<-SQL
      DROP TYPE post_statuses;
    SQL
    change_column :posts, :status, :string # Previous type
  end
end
```

#### MySQL and PostgreSQL

Running the following commands will work as expected:

    bundle exec rails db:migrate
    bundle exec rails db:rollback

We will get a column on the database that will only accept the declared values. That's the same constraint we have on our Rails app, yay!

Unfortunately, if you are still using the file **db/schema.rb** as your source of truth for generating/re-generating the database, there are a few drawbacks.

**MySQL** will give you the next result:

```ruby
  create_table "posts", options: "ENGINE=InnoDB DEFAULT CHARSET=utf8", force: :cascade do |t|
    t.string "title"
    t.text "content"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
    t.string "status", limit: 9
  end
```

As you can see, there is nothing that indicates that your **status** column is an **enum**. That's because Rails does not know anything about native **enum** column type from **MySQL**.

**PostgreSQL's** result is even worst:

```ruby
# Could not dump table "posts" because of following StandardError
# Unknown type 'post_statuses' for column 'status'
```

Yes, that's what you will get in your **db/schema.rb** when using **PostgreSQL** to create an **enum** column on a migration. Really bad, uh?

Fortunately, the solution to this problem is the same and it is really simple: Use **db/structure.sql** instead of **db/schema.rb**!

Add this line to your **config/application.rb** file:

```ruby
# config/application.rb
...
class Application < Rails::Application
  ...
  config.active_record.schema_format = :sql
  ...
end
...
```

Problem solved! Now you are safe to delete your **db/schema.rb** file and git track the brand new **db/structure.sql**. It will contain the most reliable dump of your database's structure.

## Wrapping up

ActiveRecord Enums are really useful when used correctly. We can make life easier for people with write/read access to our application's database by using a **string** column instead of an **integer** one.

As **Uncle Ben** once said:

> "With great power comes great responsibility"

We need to add the same constraints on our database, to guarantee data integrity. Database native enums are great for that and they **keep the same readability than a string column**, with the **built-in protection against bad input!**
