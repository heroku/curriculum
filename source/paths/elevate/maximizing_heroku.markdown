---
layout: page
title: Maximizing Heroku
section: Salesforce Elevate
sidebar: true
back: /elevate
---

You've gotten an application up and running, but how do you make sure it stays up and highly responsive?

## Touring the Web Interface

A poweruser is going to use Toolbelt to do most everything from the terminal. But there are a lot of things you can do in the web interface. Let's take a look at several of them including:

* scaling dyno numbers and size
* browse and manage add-ons
* display the deployment history
* control who has deployment access
* change the name, 404, domain names, and ownership

## Scaling Web Dynos

Even with threading, the traffic a single dyno can serve is limited. The easiest way to scale the number of concurrent requests your application can handle is to increase the number of dynos.

### Through the GUI

Option one is to use the graphical interface on Heroku.com. You should:

* Visit https://dashboard.heroku.com/apps
* Click the name of the application you want to manipulate
* Click "Edit" on the far right
* Under "Dynos", slide the marker to the right
* Click "Save" (your account will really be debited by the minute)

### From the CLI

Scaling through the GUI is cute, but it's not fast and difficult to script. Maybe, for instance, you find that your traffic spikes during certain hours and want to scale up your dynos *just for those hours*. You could write a script to manipulate the dynos from the command line.

Within your project directory you can check on your current dynos:

{% terminal %}
$  heroku ps
=== web (1X): `target/start -Dhttp.port=${PORT} ${JAVA_OPTS} -DapplyEvolutions.default=true -Ddb.default.driver=org.postgresql.Driver -Ddb.default.url=${DATABASE_URL}`
web.1: up 2014/03/24 19:53:41 (~ 25m ago)
{% endterminal %}

From that we know:

* `web (1X)` shows that we're using Heroku's smallest, cheapest dyno type
* `target/start` is the actual command that is executed on the dyno
* `web.1` tells us that only a single dyno is running

From there you can do everything available in the GUI, such as changing the dyno type:

{% terminal %}
$ heroku ps:resize web=2X
Resizing and restarting the specified dynos... done
web dynos now 2X ($0.10/dyno-hour)
{% endterminal %}

And changing the number of dynos up to 8:

{% terminal %}
$ heroku ps:scale web=8
Scaling dynos... done, now running web at 8:2X.web dynos now 2X ($0.10/dyno-hour)
{% endterminal %}

And checking the results with `ps` again:

{% terminal %}
$ heroku ps
=== web (2X): `target/start -Dhttp.port=${PORT} ${JAVA_OPTS} -DapplyEvolutions.default=true -Ddb.default.driver=org.postgresql.Driver -Ddb.default.url=${DATABASE_URL}`
web.1: up 2014/03/20 11:50:59 (~ 1m ago)
web.2: up 2014/03/20 11:51:45 (~ 38s ago)
web.3: up 2014/03/20 11:51:44 (~ 39s ago)
web.4: up 2014/03/20 11:51:45 (~ 38s ago)
web.5: up 2014/03/20 11:51:46 (~ 38s ago)
web.6: up 2014/03/20 11:51:45 (~ 39s ago)
web.7: up 2014/03/20 11:51:45 (~ 39s ago)
web.8: up 2014/03/20 11:51:47 (~ 37s ago)
{% endterminal %}

Now your dynos are twice as powerful, and there are eight times as many of them! Get back to the free settings with:

{% terminal %}
$ heroku ps:resize web=1X
Resizing and restarting the specified dynos... done
web dynos now 1X ($0.05/dyno-hour)
$ heroku ps:scale web=1
web dynos now 1X ($0.05/dyno-hour)
$ heroku ps
=== web (1X): `target/start -Dhttp.port=${PORT} ${JAVA_OPTS} -DapplyEvolutions.default=true -Ddb.default.driver=org.postgresql.Driver -Ddb.default.url=${DATABASE_URL}`
web.1: up 2014/03/20 11:55:08 (~ 57s ago)
{% endterminal %}

### Scaling in Action

In the sample application we've embedded a small test script that will make several requests to your application. Run it like this:

{% terminal %}
$ java generate_requests
{% endterminal %}

And you will see output that starts like this:

{% terminal %}
Starting the test.
Request 1 responded after 2.0 seconds
Request 2 responded after 4.0 seconds
Request 3 responded after 6.0 seconds
Request 4 responded after 8.0 seconds
Request 5 responded after 10.0 seconds
{% endterminal %}

Note that the total response time is increasing linearly because the requests are queuing up for the single dyno.

#### Parallelizing with Dynos

Now let's ramp up four dynos and see what happens:

{% terminal %}
$ heroku ps:scale web=4
{% endterminal %}

And run the test again:

{% terminal %}
$ java generate_requests
Starting the test.
Request 1 responded after 2.0 seconds
Request 2 responded after 2.0 seconds
Request 3 responded after 2.0 seconds
Request 4 responded after 2.0 seconds
Request 5 responded after 4.0 seconds
{% endterminal %}

Your numbers will vary, but you will see greater variability in the total response time but *all* of them should be well under the slowest from the first run. Our requests are getting served by multiple dynos, balancing the load.

#### Scaling Memory

It turns out that the end point we're hitting makes use of a lot of RAM. Our 1X dynos are struggling. Let's scale up the memory available by using a 2X dyno and run the test again:

{% terminal %}
$ heroku ps:size web=2X
$ java generate_requests
{% endterminal %}

You should observe your responses coming back even faster.

## Using the `Procfile`

Heroku's *Cedar* stack allows you a lot of flexibility through the `Procfile`. You can define and name one or more commands that your application should run when deployed. We saw an example of a command to start the Play application earlier.

### A Basic `Procfile`

Typically this starts with a `web` server. A `web` process type for a Play application looks like this:

```plain
web: target/start -Dhttp.port=${PORT} ${JAVA_OPTS} -DapplyEvolutions.default=true -Ddb.default.driver=org.postgresql.Driver -Ddb.default.url=${DATABASE_URL}
```

* The `web` part defines the name of the process type
* The part to the right of the `:` is what you'd run from a UNIX terminal to execute the command
* Environment variables can be used (like `$PORT` and `$RACK_ENV` here)
* Runs the command `target/start`
* Passes several options from the environment variables (`PORT`, `JAVA_OPTS`, `DATABASE_URL`)
* Automatically runs the database "evolutions" if needed

### Defining Multiple Process Types

If you want to run multiple dynos each running the same application, like eight instances of your `web` process type, then you're already done.

Commonly, however, you'll want to run multiple *different* process types. An application, for instance, might want to have 16 dynos running the `web` process to respond to web requests, then four dynos running as background workers sending email or doing other jobs.

You can define multiple process types in the `Procfile`:

```plain
web: bundle exec thin start -p $PORT -e $RACK_ENV
worker: bundle exec rake jobs:work
```

Other than `web` to respond to requests, you can makeup whatever process names are germane to your domain. Whatever name you used can be used from the web interface or CLI to scale dynos up and down.

### References

* [Procfile configuration options](http://Dev Center.heroku.com/articles/procfile) on Heroku's Dev Center

## Configuration

Web applications typically rely on some pieces of data that either can't or shouldn't be stored in the source code. This might include:

* Resource locations, like the IP address of the database
* Security credentials, like OAuth tokens
* Execution environment markers, like *"production"* and *"development"*

These bits of data can be stored on Heroku and made available as environment variables.

### Querying the Config

All apps start out with a default set of variables. From within a Heroku project's directory on your local system, run `heroku config`. The below example is from a Java Play application with some pieces removed for security:

{% terminal %}
$ heroku config
=== boiling-island-2815 Config Vars
DATABASE_URL:               postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
HEROKU_POSTGRESQL_JADE_URL: postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
JAVA_OPTS:                  -Xmx384m -Xss512k -XX:+UseCompressedOops
PATH:                       .jdk/bin:.sbt_home/bin:/usr/local/bin:/usr/bin:/bin
REPO:                       /app/.sbt_home/.ivy2/cache
SBT_OPTS:                   -Xmx384m -Xss512k -XX:+UseCompressedOops
{% endterminal %}

Note, particularly, the `DATABASE_URL` which is the location of the PostgreSQL database provisioned for this application.

### Adding a Value

Say we now want to store a key named `OAUTH_SHARED_SECRET` on the server. We use `heroku config:add`:

{% terminal %}
$ heroku config:add OAUTH_SHARED_SECRET="helloworld"
Setting config vars and restarting boiling-island-2815... done, v8
OAUTH_SHARED_SECRET: helloworld
{% endterminal %}

Then we can query for the defined values again to verify it's there:

{% terminal %}
$ heroku config
=== boiling-island-2815 Config Vars
DATABASE_URL:               postgres://zvxdqpyretrsgk:DNKuFujjxCLKoCV3qQFyH7kz_E@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
HEROKU_POSTGRESQL_JADE_URL: postgres://zvxdqpyretrsgk:DNKuFujjxCLKoCV3qQFyH7kz_E@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
JAVA_OPTS:                  -Xmx384m -Xss512k -XX:+UseCompressedOops
OAUTH_SHARED_SECRET:        helloworld
PATH:                       .jdk/bin:.sbt_home/bin:/usr/local/bin:/usr/bin:/bin
REPO:                       /app/.sbt_home/.ivy2/cache
SBT_OPTS:                   -Xmx384m -Xss512k -XX:+UseCompressedOops
{% endterminal %}

### Accessing Values in Code

Defining those pieces of data is only useful if you can access them from your code. Below are examples for both Ruby and Java. The implementation will depend on your language of choice, not Heroku. Typically if you ask for a key that is not defined you'll get back an empty string.

#### Java

In Java, accessing environment variables is as easy as calling `System.getenv` and passing the name of the key:

{% terminal %}
$ heroku run sbt play console
Running `sbt play console` attached to terminal... up, run.2176
Picked up JAVA_TOOL_OPTIONS:  -Djava.rmi.server.useCodebaseOnly=true
Getting org.scala-tools.sbt sbt_2.9.1 0.11.2 ...
...
scala> System.getenv("OAUTH_SHARED_SECRET")
res0: java.lang.String = helloworld
{% endterminal %}

### Considering Security

Environment variables are an appropriate place to store secure credentials, but you must keep in mind who has read access to them. Any of the following people could read your environment variables:

* Collaborators on an application
* A person who steals your laptop and can login as you and you have no SSH passphrase
* A person who accesses your computer while the SSH keychain is unlocked

If that's a concern, then you can mitigate the issue by reducing deployment access. For instance, you could setup a Continuous Integration server which runs your tests and, if they pass, it deploys the code. The majority of developers wouldn't need access to the Heroku application itself, so there's less risk.

### References

* [Configuration Variables](http://Dev Center.heroku.com/articles/config-vars) on Heroku's Dev Center

## Installing an Add-on / Upgrading Your Database

One of Heroku's great strengths is the rich library of add-ons. There are dozen of options available at https://addons.heroku.com/ , giving you everything from data storage to video processing. Many of them can be installed/setup with little or no change to your application code.

Heroku offers [many different data storage options](https://addons.heroku.com/#data-stores), but most applications are centered around PostgreSQL. Let's look at how to upgrade to a production-tier Postgres instance using the add-ons system.

### PostgreSQL Levels

When you provision an application on Heroku it automatically comes with the **Hobby-Dev** level database.

#### Hobby-Dev Limitations

You should be aware that the free Hobby-Dev database has some significant limitations:

* It allows only 10,000 rows of data in aggregate across all tables
* It does not allow for "follower" databases, making backup more complex
* Max 20 connections
* 0MB of reserved, dedicated RAM

#### Standard-0

We'll upgrade to the Standard-0 level which:

* Costs $50/month
* Unlimited data rows, 64gb total data
* Follower databases enabled
* Max 120 connections
* 1GB RAM

From there it goes up and up. You can spend $6,000/month on an instance with 120GB of dedicated RAM and support for a terabyte of data. You can [see all the options here](https://www.heroku.com/pricing).

### Replacement vs Migration

Let's look at how to upgrade an application from the "Hobby Dev" to "Standard 0", the bottom level that Heroku considers "production scale."

There are different procedures for replacing the database with a new one versus migrating the existing data to a new instance. Let's look at the easier of the two, full replacement.

### Checking the Before-State

Before we start changing things around, let's look at the existing application configurations `DATABASE_URL` and `HEROKU_POSTGRESQL` keys:

{% terminal %}
heroku config
=== boiling-island-2815 Config Vars
DATABASE_URL:               postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
HEROKU_POSTGRESQL_JADE_URL: postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
{% endterminal %}

When we add a new database that `JADE` URL will stick around. This allows us to connect to the old database, in this case a free instance, if we wanted to access old data.

If, however, the old database were a paid plan, we'd keep getting charged until it's deprovisioned.

### Using `pg:info`

The key name is a random color and doesn't tell you anything about what actual database type is at that address. For much more useful information, we can use `pg:info`:

{% terminal %}
$ heroku pg:info
=== HEROKU_POSTGRESQL_JADE_URL (DATABASE_URL)
Plan:        Dev
Status:      available
Connections: 5
PG Version:  9.3.3
Created:     2014-03-26 03:19 UTC
Data Size:   6.7 MB
Tables:      3
Rows:        574/10000 (In compliance)
Fork/Follow: Unsupported
Rollback:    Unsupported
{% endterminal %}

If your application had more than one PostgreSQL database then there'd be more than one listing here.

### Provisioning

To add the new database instance we just need a single instruction:

{% terminal %}
$ heroku addons:add heroku-postgresql:standard-0
Adding heroku-postgresql:standard-0 on boiling-island-2815... done, v9 ($50/mo)
Attached as HEROKU_POSTGRESQL_ROSE_URL
The database should be available in 3-5 minutes.
 ! The database will be empty. If upgrading, you can transfer
 ! data from another database with pgbackups:restore.
Use `heroku pg:wait` to track status.
Use `heroku addons:docs heroku-postgresql` to view documentation.
{% endterminal %}

As instructed, you can run `heroku pg:wait` which will hang until the new instance is ready.

{% terminal %}
$ heroku pg:wait
Waiting for database HEROKU_POSTGRESQL_ROSE_URL... available
{% endterminal %}

Now you're ready to actually use it.

### Configuring

Your application, either through the `Procfile` or code itself, should be relying on the `DATABASE_URL` environment variable. Therefore, using this database should be as easy as:

* Change the `DATABASE_URL` variable to the newly provisioned instance
* Restart the application
* Run any data migrations / evolutions

#### Change the `DATABASE_URL`

If you run `heroku config` again, you'll see the new database location defined:

{% terminal %}
$ heroku config
=== boiling-island-2815 Config Vars
DATABASE_URL:               postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
HEROKU_POSTGRESQL_JADE_URL: postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
HEROKU_POSTGRESQL_ROSE_URL: postgres://username:password@ec2-54-83-63-243.compute-1.amazonaws.com:5542/d4bibagdniev3f
{% endterminal %}

Then `unset` the `DATABASE_URL`...

{% terminal %}
$ heroku config:unset DATABASE_URL
Unsetting DATABASE_URL and restarting boiling-island-2815... done, v10
{% endterminal %}

And set it using the value of `HEROKU_POSTGRESQL_ROSE_URL` from above:

{% terminal %}
$ heroku config:set DATABASE_URL=postgres://username:password@ec2-54-83-63-243.compute-1.amazonaws.com:5542/d4bibagdniev3f
Setting config vars and restarting boiling-island-2815... done, v11
DATABASE_URL: postgres://username:password@ec2-54-83-63-243.compute-1.amazonaws.com:5542/d4bibagdniev3f
{% endterminal %}

#### Migrating / Evolving

At this point you've got a blank database. If you're deploying a Rails application, you'd want to run your migrations:

{% terminal %}
$ heroku run bundle exec rake db:migrate
{% endterminal %}

If you're running a Play application with auto-apply evolutions enabled, then they'll be run on the first request.

### Deprovisioning

**Please carefully think through what you're doing before following these instructions.** If you deprovision a database on Heroku you cannot get the data back.

That production-quality instance we just added upped our bill by $50/month. That far exceed the budget of a little sample application. Let's undo it:

* Change the `DATABASE_URL` back to the free instance
* Remove the Standard-0 instance

It's the reverse of what we did before:

{% terminal %}
$ heroku config:unset DATABASE_URL
$ heroku config:set DATABASE_URL=postgres://username:password@ec2-54-225-101-119.compute-1.amazonaws.com:5432/d4tstdg3etpui8
$ heroku addons:remove heroku-postgresql:standard-0
{% endterminal %}

Where the URL in step two was our original `HEROKU_POSTGRESQL_JADE_URL`.

The add-on removal will ask you for a confirmation. **Consider** that a person who has access to your application could similarly *drop the production database*.

### References

* [Choosing the Right Heroku PostgreSQL Plan](https://Dev Center.heroku.com/articles/heroku-postgres-plans#hobby-tier)
* [Creating and Managing Postgres Follower Database](https://Dev Center.heroku.com/articles/heroku-postgres-follower-databases)

## Setting Up Custom Domains

Heroku's haiku-inspired generated URLs like *"boiling island"* are cute, but most of the time you'll want to use a custom domain you've purchased elsewhere.

### Adding a Domain Name

Add the domain names like this:

{% terminal %}
$ heroku domains:add www.example.com
$ heroku domains:add example.com
{% endterminal %}

Note that, yes, you need both if you want both http://example.com and http://www.example.com to resolve to your application.

### DNS Configuration

Your DNS settings are likely controlled by the registrar you used to purchase the domain (like http://dnsimple.com or http://namecheap.com). You've configured Heroku to **listen** for requests to those domains, but you need to configure the DNS server to **send** the traffic to Heroku in the first place.

The exact settings and process will vary per registrar, but essentially you want to create a CNAME record pointing to your application's `herokuapp` URL, like `boiling-island-2815.herokuapp.com`.

### References

* [Custom Domains](https://Dev Center.heroku.com/articles/custom-domains) on Heroku's Dev Center
