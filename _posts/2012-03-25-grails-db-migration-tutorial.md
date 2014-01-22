---
layout: post
category : tutorials
tags : [grails, tutorial, database, migration]
---

I've used Grails for a few projects recently, and, while I've enjoyed the framework as a whole, there have been two major pain points coming from my experiences with Spring MVC and rails. These are:

*   Testing (Quick note: I use [spock][1] for my grails testing)
*   database migrations

With the release of Grails 2.0.0 and the official db-migration plugin, the latter has become much less of a problem, but it still seems to suffer from some lack of good documentation and examples. So with this post, I'll try to walk through an example of installing this plugin to an existing Grails project and making some domain model changes.

## Setup / Configuration

Quick note: the source for the example project that I'll be referencing here is [available on GitHub][2]. If you want to follow along, the commit history builds up piece by piece along with this post.

First things first: we need to install the plugin for the project. I'll be using the cliché theoretical Blog example here. Navigate to the project directory and install the plugin.

{% highlight bash %}
blog$ grails install-plugin database-migration
| Plugin installed.
{% endhighlight %}

As of this writing, this will install version 1.0 of the plugin (as evidenced by the updated application.properties of the project).

{% highlight properties %}
app.grails.version=2.0.0
app.name=blog
app.servlet.version=2.5
app.version=0.1
plugins.database-migration=1.0
{% endhighlight %}

Now, since in this example we are adding the plugin to an established project, we need to change a bit of configuration to make sure the db-migrations plugin will handle all database changes.

You will need to remove any “dbCreate” lines from grails-app/conf/DataSource.groovy. This is done to prevent grails from attempting to make changes to the schema on its own. If this step is skipped, errors can crop up with db-migration where a migration will attempt to add a table or column that was already added by grails. Here is an example with a comment on the lines that should be deleted.

{% highlight groovy %}
// environment specific settings
environments {
  development {
    dataSource {
      url = "jdbc:h2:mem:devDb;MVCC=TRUE"
      dbCreate = "create-drop" //Delete me
    }
  }
  test {
    dataSource {
      url = "jdbc:h2:mem:testDb;MVCC=TRUE"
      dbCreate = "update" //Delete me
    }
  }
}
{% endhighlight %}


However, in my case, I use a configuration in the .grails folder of my home directory to configure my project data source, so the line needs to be removed from **this** file. This step is *only* if you use this type of configuration. In ~/.grails/blog-config.groovy:

{% highlight groovy %}
dataSource {
    dbCreate = "update" //Delete me
    url = "jdbc:postgresql:blog"
    pooled = true
    dialect = org.hibernate.dialect.PostgresDialect
}
{% endhighlight %}

###Baseline

Now we will capture a baseline of our database that migrations will be added to. To do this, we will read the current state of the classes and write it into our baseline changeset.

{% highlight bash %}
blog$ grails dbm-generate-gorm-changelog changelog.groovy
{% endhighlight %}

This generates a changelog file based on the gorm domain classes and stores it in grails-app/migrations/changelog.groovy. A quick peek at this file shows that it is comprised of a series of changeSets written in a Groovy DSL ([Domain Specific Language][3]) as opposed to raw SQL. When we get to the point of creating migrations, they will essentially look identical.

Here is a snippet from the changelog.groovy that represents the current Post domain object with a title and body. I've also included the corresponding domain class for reference.

{% highlight groovy %}
databaseChangeLog = {

    changeSet(author: "wpgreenway (generated)", id: "1336436810786-1") {
        createTable(tableName: "post") {
            column(name: "id", type: "int8") {
                constraints(nullable: "false", primaryKey: "true", primaryKeyName: "postPK")
            }

            column(name: "version", type: "int8") {
                constraints(nullable: "false")
            }

            column(name: "body", type: "text")

            column(name: "title", type: "varchar(255)")
        }
    }

    changeSet(author: "wpgreenway (generated)", id: "1336436810786-2") {
        createSequence(sequenceName: "hibernate_sequence")
    }
}
{% endhighlight %}

{% highlight groovy %}
class Post {

    String title
    String body

    static mapping = {
        body type: 'text'
    }

    static constraints = {
        title(nullable: true)
        body(nullable: true)
    }
}
{% endhighlight %}

Finally, we want to update the new databasechangelog table to capture the fact that the contents of the changelog.groovy represent the existing schema of our database. To do this, run the following command.

{% highlight bash %}
grails dbm-changelog-sync
{% endhighlight %}

This will insert rows into the databasechangelog for the changeSets defined in the changelog and mark them as “EXECUTED”.

With that, we are ready to start writing migrations and updating our database!

## Making a Change

So we have a “blog” project that just has a simple Post domain class that describes a single post. When the class was written it had a title and body, but it seems like a good idea to keep track of *who* wrote each post as well. To have a simple solution for the purposes of this example, we will just add an author String to the class (as opposed to linking to a users table as would likely be the case in a real-world solution). To keep things simple, we will allow this field to be null for anonymous postings. With the new field added the class looks like this:

{% highlight groovy %}
class Post {

    String title
    String body
    String author

    static mapping = {
        body type: 'text'
    }

    static constraints = {
        title(nullable: true)
        body(nullable: true)
        author(nullable: true)
    }
}
{% endhighlight %}

## Generating a Migration

With that file saved, we can generate a changeset in the same DSL that we saw in the changelog.groovy from before using the dbm-gorm-diff command. This command compares the gorm classes with the database and writes the differences to the specified file. So let's run it with our change and put it in file with a descriptive name.

{% highlight bash %}
blog$ grails dbm-gorm-diff add-author-to-post.groovy -add
{% endhighlight %}

This writes the following file to grails-app/migrations/add-author-to-post.groovy:

{% highlight groovy %}
databaseChangeLog = {

    changeSet(author: "wpgreenway (generated)", id: "1336437476874-1") {
        addColumn(tableName: "post") {
            column(name: "author", type: "varchar(255)")
        }
    }
}
{% endhighlight %}

So now we have a file that represents our migration, but this won't necessarily be recognized automatically. You'll notice that when running that command, we appended the -add option. This will automatically add a line to the end of the master changelog.groovy that will include the newly generated change. If a migration is created without that option, just manually add a line like the following to the end of grails-app/migrations/changelog.groovy:

{% highlight groovy %}
include file: 'add-author-to-post.groovy'
{% endhighlight %}

The changelog.groovy is always run from beginning to end, so make sure that you always add newly created migrations to the end.

## Migrating the Database

So the domain has changed and the migration is ready to go. Now it's time to actually update the database. To do this we will use the dbm-update command. This command updates the database to the latest version and will run all of the changesets that have not been executed.

{% highlight bash %}
blog$ grails dbm-update
{% endhighlight %}

So now when we describe our database we should see the updated post table schema. Awesome! Our first painless database migration is complete.

## Bonus Round: Rollback

Since our change didn't have any custom SQL, grails will be able to automatically determine how to revert the change in a rollback. If we decide we need to undo the most recent change, it's just a matter of running a single command:

{% highlight bash %}
blog$ grails dbm-rollback-count 1
{% endhighlight %}

This will rollback 1 (our most recent) change. You can give larger numbers to go back multiple steps. Additional documentation is available on the [database migration GitHub page][4]

## Not Always Quite So Simple

In any project, the database schema is sure to change, and those changes won't always be as simple as&nbsp;adding a new nullable column. Next, we will cover a slightly more complex example (that can easily be expanded to cover many types of migrations). So now that we've covered setup and a very basic migration, it's time to look at a common migration pattern that requires a bit more.

## The Change

So after our last change, we have a blog post that has a title, body, and author. The piece of information that we forgot, though, is ***when*** the post was written. Unlike the author field, this is something that we want to be a required attribute for every post, so we don't want any null values for the new column in the database. So now we'll add this new date to our domain object. With the new field added the class looks like this:

{% highlight groovy %}
class Post {

    String title
    String body
    String author
    Date dateCreated

    static mapping = {
        body type: 'text'
    }

    static constraints = {
        title(nullable: true)
        body(nullable: true)
        author(nullable: true)
    }
}
{% endhighlight %}

A couple of things to note about this change:

First, notice that the variable we added is “dateCreated”. This is important, because that is a magic field name that tells grails that it should automatically populate it with the current date/time when the record is created. That way, we aren't responsible for writing any code to populate this field, when someone clicks “Post” it will automatically happen.

Second, since we did not add a constraint for this field stating that it is nullable, it will be required for all records. This is the intended behavior, but as we will see, it will have some bearing on how our migration is written.

## Writing a Migration by Hand

Previously, we used the dbm-gorm-diff command to generate a migration. This time we'll focus on writing one from scratch. So we want to add a new column like in part 2, but we also want to add a not null constraint. So we write the following in a new file: grails-app/migrations/add-date-created-to-post.groovy

{% highlight groovy %}
databaseChangeLog = {

    changeSet(author: "wpgreenway", id: "add-date-created-to-post") {
        addColumn(tableName: "post") {
            column(name: "date_created", type: "timestamp") {
                constraints nullable: false
            }
        }
    }
}
{% endhighlight %}


You'll notice that I've made given the changeSet a descriptive ID as well, instead of just a numeric one like the generated one before did. All the plugin cares about is that each changeset has a unique ID per author so that it can keep track of what has been applied. And giving it a descriptive name can help others who might maintain the script later.

And since we didn't generate this migration with dbm-gorm-diff with the -add option, a reference to this file also needs to be added to the master changelist. So add a line like the following to the end of grails-app/migrations/changelog.groovy:

{% highlight groovy %}
include file: 'add-date-created-to-post.groovy'
{% endhighlight %}

It almost seems too easy! So let's run the migration:

{% highlight bash %}
blog$ grails dbm-update
{% endhighlight %}

And it **failed** with a big stacktrace! What happened? In my case it boiled down to the following error:

{% highlight bash %}
Caused by PSQLException: ERROR: column "date_created" contains null values
{% endhighlight %}

So since we already have data, the not-null constraint is preventing us from adding the new column. What to do?

## Multiple Step Migration

Since there have already been posts made before this migration, they do not have a value for the date_created field in the database. Since this violates a constraint, we have to make sure that they have a value before we can get this migration to run. Again, to keep things simple we'll say that it's easiest to just assume that all posts up to this point will have a post time of right now, and we'll just focus on keeping track of that information moving forward. So the sequence of events will need to be:

1.  Create a new column (allowing null values)
2.  Make sure all table rows have a value for the new column
3.  Implement the not-null constraint

Luckily, the db-migration scripts allow us to execute arbitrary SQL in addition to the commands like addColumn(). We'll leverage this to accomplish step 2.

The script ends up looking like this:

{% highlight groovy %}
databaseChangeLog = {

    changeSet(author: "wpgreenway", id: "add-date-created-to-post") {
        addColumn(tableName: "post") {
            column(name: "date_created", type: "timestamp")
        }

        grailsChange {
            change {
                sql.execute("UPDATE post SET date_created = NOW()")
            }
            rollback {
            }
        }

        addNotNullConstraint(tableName: "post", columnName: "date_created")
    }
}
{% endhighlight %}

This demonstrates a couple of new capabilities of the migration script. First, the grailsChange{} block allows us to give SQL statements for the forward change as well as the rollback. In this example, there is no code required for the rollback since the column would be removed entirely. For other migrations, this is where you would write SQL to revert the table back to the state prior to the migration. Finally, we make use of the addNotNullConstraint() with our table and column to add a constraint after new values have been assigned and it is valid.

Run this migration and hold your breath…

{% highlight bash %}
blog$ grails dbm-update
{% endhighlight %}

And now it **works!** Now we have a date that will automatically be populated for any new posts that we create.

## Final Notes

It's true that this isn't the most complicated example, but it demonstrates how to define complicated changes and rollbacks using SQL directly. Now, I will note that it is possible to use GORM for these changes as well. It would look more like this:

{% highlight groovy %}
grailsChange {
  change {
    Post.list().each { post ->
      post.dateCreated = new Date()
      post.save(failOnError: true, flush: true)
    }
  }
}
{% endhighlight %}

But this can be ***dangerous***. There is a [great blog post][5] that outlines these dangers with some specific examples. It's up to you to determine what is best for your project, but I'd definitely suggest utilizing SQL to avoid complications.

## Simple but Effective

Even with the simple tools outlined here, I've been able to tackle the majority of the changes that happen to my domain classes in Grails projects. I welcome any suggestions to improve this process.

Hopefully this was helpful. If so, remember that you can view or download [the project source][6] for future reference.


 [1]: http://code.google.com/p/spock/ "Spock"
 [2]: https://github.com/wpgreenway/grails-blog-example "Example Project Repository"
 [3]: http://en.wikipedia.org/wiki/Domain-specific_language "Domain Specific Language"
 [4]: http://grails-plugins.github.com/grails-database-migration/docs/manual/ref/Rollback%20Scripts/dbm-rollback-count.html "Database Migration Documentation"
 [5]: http://refactr.com/blog/2012/01/grails-database-migration-gotchas/
 [6]: https://github.com/wpgreenway/grails-blog-example