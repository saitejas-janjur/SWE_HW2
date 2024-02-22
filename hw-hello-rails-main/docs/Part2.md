# Part 2

## Database in different environments

You wouldn't want to develop or test your app against the production database, as bugs in your code might accidentally damage valuable customer data. For that reason, Rails defines three _environments_ -- `production`, `development`, and `test` -- each of which manages its own separate database. These environments, and the means for connecting to the database associated with each, are stored by Rails in  `config/database.yml`.

The `test` database is entirely managed by the testing tools and should never be modified manually: it should be wiped clean and repopulated at the beginning of every testing run.

## Create the database

A brand-new Rails app has no database, so we need to create one. The default `config/database.yml`, which we will use, specifies that the development database will be stored in the file `db/development.sqlite3`. We could use the  [SQLite3 command-line tool](http://www.sqlite.org/cli.html) or a SQLite GUI tool to create it manually, but how would we later create the database and table in our production database when we deploy?  Typing the same commands a second time isn't DRY, and the exact commands might be hard to remember. Further, if the production database is something other than SQLite3 (as is almost certainly the case), the specific commands might be different. And in the future, if we add more tables or make other changes to the database, we'll face the same problem.

## Create and apply the migration

A better alternative is a **migration**, a tool provided by Rails. It is a portable script that changes the database schema (layout of tables and columns) in a consistent and repeatable way, just as Bundler uses the Gemfile to identify and install necessary gems (libraries) in a consistent and repeatable way. Changing the schema using migrations is a four-step process:

1. Create a migration describing what changes to make. As with `rails new`, Rails provides a migration _generator_ that gives you the boilerplate code, plus various helper methods to describe the migration.
2. Apply the migration to the development database.  Rails defines a `rake` task for this. `rake` (i.e. Ruby make) is a popular task runner for Rails and contains many predefined tasks, including applying a migration. You can run rake tasks with `rake` or `rails`.  The convention when using Ruby on Rails is to run rake tasks using `rails`.
3. Assuming the migration succeeded, update the test database's schema by running `rails db:test:prepare`.
4. Run your tests, and if all is well, apply the migration to the production database and deploy the new code to production.  The process for applying migrations in production depends on the deployment environment; we'll do that for Heroku in a later part of the exercise.

We'll use the first 3 steps of this process to add a new table that stores each movie's title, rating, description, and release date. Each migration needs a name, and since this migration will create the movies table, we choose the name `create_movies`. Run the command:

```sh
rails generate migration create_movies
```

If successful, you will find a new file under `db/migrate` whose name begins with the creation time and date and ends with the name you supplied, for example, `20230816015503_create_movies.rb`. (This naming scheme lets Rails apply migrations in the order they were created, since the file names will sort in date order.)

Paste the code below into the file and save it.

```ruby
class CreateMovies < ActiveRecord::Migration[7.0]
  def change
    create_table 'movies' do |t|
      t.string 'title'
      t.string 'rating'
      t.text 'description'
      t.datetime 'release_date'
      # Add fields that let Rails automatically keep track
      # of when movies are added or modified:
      t.timestamps
    end
  end
end
```

As you can see, migrations illustrate an idiomatic use of blocks: the `ActiveRecord::Migration#create_table`  method takes a block of 1 argument and yields to that block an object representing the table being created.  The methods `string`, `datetime`, and so on are provided by this table object, and calling them results  in creating columns in the newly-created database table; for example, `t.string 'title'` creates a column  named `title` that can hold a string, which for most databases means up to 255 characters. The documentation for the `ActiveRecord::Migration` class (from which all migrations inherit) is part of the [Rails documentation](http://api.rubyonrails.org/), and gives more details and other migration options.

Run

```sh
rails db:migrate
```

to actually apply the migration and create this table.  Note that this housekeeping task also stores the migration number itself in the database, and by default it only applies migrations that haven't already been applied.  (Type `rails db:migrate` again to verify that it does nothing the second time.)

## Create the initial model, and seed the database

Although the database now contains a `movies` table, Rails has no idea that it exists. Since Movies are a model, the next step is therefore to create the ActiveRecord model that uses this table.  That is as simple as creating (and versioning!!) a file `app/models/movie.rb` with these two lines:

```ruby
class Movie < ActiveRecord::Base
end
```

Recall that convention over configuration says that Rails expects the model named `Thing` to be defined in `app/models/thing.rb` and to correspond to a database table `things`.  (Yes, Rails does the right thing pluralizing irregular nouns, but to keep things simple, we recommend not using them!  Do you really need a table named `sheep`?)

You can now verify that `Movie` is defined as a model by running `rails console`, which loads up your entire app into a REPL (read-eval-print loop) environment, and executing `Movie.new`.  You will get a brand new instance of `Movie` whose attribute values are all `nil`, and importantly, whose database primary key `id` is `nil` because it hasn't been saved to the database yet.  You can confirm the latter by executing `Movie.first`, which tries to fetch the very first movie in the database; you should get back `nil`.

Now that you've done it the hard way, you should know that you can generate the model and the migration all at once using: 

```sh
rails generate model Movie title:string rating:string description:text release_date:datetime
```

As a last step before continuing, you can now _seed_ the database (i.e. add initial data) with some movies to make the rest of the activity more interesting. Copy the code below into `db/seeds.rb`:

```ruby
# Seed the RottenPotatoes DB with some movies.
more_movies = [
  {:title => 'My Neighbor Totoro', :rating => 'G',
    :release_date => '16-Apr-1988'},
  {:title => 'Green Book', :rating => 'PG-13',
    :release_date => '16-Nov-2018'},
  {:title => 'Parasite', :rating => 'R',
    :release_date => '30-May-2019'},
  {:title => 'Nomadland', :rating => 'R',
    :release_date => '19-Feb-2021'},
  {:title => 'CODA', :rating => 'PG-13',
    :release_date => '13-Aug-2021'}
]

more_movies.each do |movie|
  Movie.create!(movie)
end
```

Once you have a sense of what it does, run it by executing:

```sh
rails db:seed
```

Now once again open the rails console and execute `Movie.first` and verify that there are movies in the database.  In fact, using the [ActiveRecord Basics CHIPS](https://github.com/saasbook/hw-activerecord-practice) as inspiration, try a few simple queries on movies from the Rails console.


## Summary

*  Rails defines three environments---`development`, `production`, and `test`---each with its own copy of the database.

*  A migration is a script describing a specific set of changes to the database.  As apps evolve and add features, migrations are added to express the database changes required to support those new features.

*  Changing a database using a migration takes three steps: create the migration, apply the migration to your development database, and (if applicable) after testing your code apply the migration to your production database.

*  The `rails generate migration` generator fills in the boilerplate for a new migration, and the `ActiveRecord::Migration` class contains helpful methods for defining it.

* The `rails generate model` generator will generate the model and the migration all at once.  You can use [rails-generate.com](https://rails-generate.com/) to generate the generator command.

* `rails db:migrate` applies  only those migrations not already applied to the development database. The method for applying migrations to a production database depends on the deployment environment.

* `rails db:seed` runs the `db/seeds.rb` file, which can optionally contain some initial data to put into the database.


<details>
<summary>
Do Rails models acquire the methods <code>where</code> and <code>find</code> via (a) inheritance or (b) mix-in?  (Hint: check the <code>movie.rb</code> file.)

</summary>
<blockquote>
(a) they inherit from <code>ActiveRecord::Base</code>.
</blockquote>
</details>

## Next
[Part 3](Part3.md)
