# Many To Many Associations

## Learning Goals

- Establish the many-to-many (or **has many through**) association in Active
  Record

## Introduction

In the previous lesson, we saw how to create a **one-to-many** association
between two models using Active Record by following certain naming conventions
and using the right foreign key on our tables when generating the migrations.

In the SQL section, we learned about one other kind of relationship: the
**many-to-many**, also known as the **has many through**, relationship. For
instance, in a domain where a **cat** has many **owners** and an **owner** can
also have many **cats**, we needed to create another table to join between those
two tables:

![Pets Database ERD](https://curriculum-content.s3.amazonaws.com/phase-3/sql-table-relations-creating-join-tables/cats-cat_owners-owners.png)

In this lesson, we'll learn how to create a **many-to-many** relationship in
Active Record. We'll continue working on our games and reviews domain, but this
time we'll add a third model into the mix: a users model. We'll be setting up these relationships:

- A game **has many** reviews
- A game **has many** users, **through** reviews
- A review **belongs to** a game
- A review **belongs to** a user
- A user **has many** reviews
- A user **has many** games, **through** reviews

Once we're done setting up the database tables, here's what the ERD will look like:

![Game Reviews ERD](https://curriculum-content.s3.amazonaws.com/phase-3/active-record-associations-many-to-many/games-reviews-users-erd.png)

To get started, run `bundle install`, then follow along with the code.

## Creating a Join Table

Right now, we've got code for the `Game` model (and the `games` table), along
with the code for the `Review` model (and the `reviews` table) from the previous
lesson.

To start, let's add the code we'll need for the `User` model as well. We'll start
by generating the migration:

```console
$ bundle exec rake db:create_migration NAME=create_users
```

Let's create the `users` table with a `name` column:

```rb
class CreateUsers < ActiveRecord::Migration[6.1]
  def change
    create_table :users do |t|
      t.string :name
      t.timestamps
    end
  end
end
```

We'll also need to modify the `reviews` table and add a foreign key to refer to
our `users` table. Remember, each review now **belongs to** a specific user. Any
time we create a **belongs to** relationship, we need a foreign key to establish
this relationship. Let's go ahead and write a migration to update the `reviews`
table:

```console
$ bundle exec rake db:create_migration NAME=add_user_id_to_reviews
```

We'll use the `add_column` method to update the `reviews` table and add a
`user_id` foreign key:

```rb
class AddUserIdToReviews < ActiveRecord::Migration[6.1]
  def change
    add_column :reviews, :user_id, :integer
  end
end
```

With that, our migrations are good to go! Run the new migrations to update
the database and schema:

```console
$ bundle exec rake db:migrate
```

Run the seed file as well to populate the `games` and `reviews` tables:

```console
$ bundle exec rake db:seed
```

## Setting Up the Join Class

Now that we've updated the database, we can start working on updating our Active
Record models. The first one we'll work on is the `Review` model. We want our
model's code to reflect the change we made in the database, so that we can
easily access data about which user left a review. Our `Review` model currently
looks like this:

```rb
class Review < ActiveRecord::Base
  belongs_to :game
end
```

We now also want our review to know which user it belongs to, so let's add that
code as well:

```rb
class Review < ActiveRecord::Base
  belongs_to :game
  belongs_to :user
end
```

Now, we can create a new `Review` instance and associate it with a `User`
**and** a `Game`. Run `rake console` and try it out:

```rb
# Get a game instance
game = Game.first
# Create a User instance
user = User.create(name: "Liza")
# Create a review that belongs to a game and a user
review = Review.create(score: 8, game_id: game.id, user_id: user.id)
```

Just like in the previous lesson, we can access data from the review instance
about the associated game; but now, we can also access data about the associated user:

```rb
review.game
# => #<Game:0x00007ff71a25f5d0 id: 1, title: "Diablo", genre: "Visual novel", ...>
review.user
# => #<User:0x00007ff71a26fe58 id: 1, name: "Liza", ...>
```

In Active Record parlance, we refer to this `Review` class as a "join" class,
because we use it to join between two other classes in our application: the
`Game` class and the `User` class. We need this association set up first before
we'll be able to access data about the users directly from their games, and
access data about the games directly from their users.

## Creating a Many-to-Many Association with `has_many through`

Let's start with the `Game` class. Here's what it looks like right now:

```rb
class Game < ActiveRecord::Base
  has_many :reviews
end
```

As a reminder, when we use the `has_many` macro, Active Record generates
an instance method `#reviews` that we can call on a `Game` instance to access
all the associated reviews:

```rb
game = Game.first
game.reviews
# => [#<Review:0x00007ff71926dac8 id: 1, ...>, #<Review:0x00007ff71926d960 id: 2, ...>
```

However, if you'll recall, we updated our tables to support another
relationship:

- A game **has many** users, **through** reviews

What this means for us in code is that it might be convenient to access a list
of all the users who left reviews for a specific game from the game instance
itself. In other words, it would be nice to be able to use a method like this
to see all the users associated with a specific game:

```rb
game.users
# => [#<User>, #<User>]
```

Writing the SQL out to access this relationship would be a bit of a pain; we'd
need to join the `reviews` table in order to access the correct users for a
specific game:

```sql
SELECT "users".*
FROM "users"
INNER JOIN "reviews"
  ON "users"."id" = "reviews"."user_id"
WHERE "reviews"."game_id" = 1
```

Luckily for us, Active Record's `has_many` macro also can be used to establish
this relationship and write that SQL for us! Here's how we can use it:

```rb
class Game < ActiveRecord::Base
  has_many :reviews
  has_many :users, through: :reviews
end
```

By adding this second `has_many` macro, and using the `through:` option, we're
now able to use that `#users` instance method with our games. Try it out
(remember to exit your console and re-start it after updating your `Game`
class):

```rb
game = Game.first
game.users
# => [#<User:0x00007f96813a5d58 id: 1, name: "Liza", ...>]
```

We can now use Active Record to go **through** the join model, `Review`, from
the `Game` model, to return the associated users, all without writing any SQL
ourselves. Pretty cool!

There are a couple important things to note when using the `has_many` macro with
the `through:` option. Order matters — you must place the first `has_many` that
references the join table **above** the second `has_many` that goes through that
join table. This code won't work:

```rb
class Game < ActiveRecord::Base
  has_many :users, through: :reviews
  has_many :reviews
end
```

Active Record won't know how to go `through` the `reviews` table until you
create the `has_many :reviews` association.

Also, these are still just Ruby methods, so it might help to see them written
out with parentheses to understand the syntax:

```rb
class Game < ActiveRecord::Base
  has_many(:reviews)
  has_many(:users, through: :reviews)
end
```

When calling `has_many`, we're passing in a first argument of a symbol that
refers to the **table name** in our database (`:users`). In the second argument,
we're passing a key-value pair, where the key is the `through` option, and the
value is the `:reviews` symbol, which refers to the `#reviews` method from the
first `has_many`. Phew!

While we're at it, we can also set up the inverse relationship:

- A user **has many** reviews
- A user **has many** games, **through** reviews

This will give us the ability to access all reviews for a particular user, as
well as all the games a particular user has reviewed. The code will look similar
to what we added to the `Game` model. Update the `User` class in
`app/models/user.rb` with the following code:

```rb
class User < ActiveRecord::Base
  has_many :reviews
  has_many :games, through: :reviews
end
```

Now, in the console, you can access a review for a user, as well as a list of
the games they have reviewed:

```rb
user = User.first
user.reviews
# => [#<Review:0x00007fc2a2ac01b8 id: 147, score: 8, ...>]
user.games
# => [#<Game:0x00007fc2a2b53710 id: 1, title: "Diablo", genre: "Visual novel", ...>]
```

Success! All of our models are now associated correctly, and have methods
available that make it convenient for us to access data across multiple database
tables using the primary key/foreign key relationship. Our Ruby code now reflects
the associations we established:

![Game Reviews ERD](https://curriculum-content.s3.amazonaws.com/phase-3/active-record-associations-many-to-many/games-reviews-users-erd.png)

## Conclusion

The power of Active Record all boils down to understanding database
relationships and following certain naming conventions. By leveraging
"convention over configuration", we're able to quickly set up complex
associations between multiple models with just a few lines of code.

The **one-to-many** and **many-to-many** relationships are the most common when
working with relational databases. You can apply the same concepts and code we
used in this lesson to any number of different domains, for example:

```txt
Driver -< Ride >- Passenger
Doctor -< Appointment >- Patient
Actor -< Character >- Movie
```

The code required to set up these relationships would look very similar to the
code we wrote in this lesson.

By understanding the conventions Active Record expects you to follow, and how
the underlying database relationships work, you have the ability to model all
kinds of complex, real-world concepts in your code!

## Resources

- [Active Record Associations][ar-associations]
- [belongs_to methods][]
- [has_many methods][]

[ar-associations]: https://guides.rubyonrails.org/association_basics.html
[belongs_to methods]: https://guides.rubyonrails.org/association_basics.html#methods-added-by-belongs-to
[has_many methods]: https://guides.rubyonrails.org/association_basics.html#methods-added-by-has-many
