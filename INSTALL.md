# StreetCRM deployment on Debian GNU/Linux

You need Python 3.4 and Python's `virtualenv` to use this.  In Debian
GNU/Linux ('Jessie' release) install these packages:

        $ sudo apt-get update
        $ sudo apt-get install apache2 git libapache2-mod-wsgi-py3 python3 python3-dev python3-setuptools python3-virtualenv virtualenv postgresql postgresql-client postgresql-server-dev-all gcc

Database
--------

For production use, we recommend PostgreSQL as the database (for development, SQLite3 is fine, and you don't need to do anything special here).  To set up PostgreSQL, first switch to the "postgres" user:

        $ su - postgres

Now create the user and db:

        $ createuser --pwprompt streetcrm
        $ createdb --owner=streetcrm streetcrm

Now we need to reload PostgreSQL:

        $ systemctl restart postgresql

You then need to add PostgreSQL to automatically start on boot:

        $ systemctl enable postgresql

The StreetCRM application
------------------------

In this document, we'll assume you're installing under `/var/www/` for
a production installation, but these instructions should be pretty
easy to adjust to any other location (e.g., your home directory for a
development installation).

First get the source tree:

        $ cd /var/www
        $ git clone https://github.com/OpenTechStrategies/streetcrm.git

Build the virtual enviroment for the website:

        $ cd streetcrm
        $ virtualenv --python=python3.4 .
        # Note: if you don't have python 3.4, check which version you
        # have by running python3 --version.  You might want to replace
        # this with `virtualenv --python=python3.5 .` or similar.
        $ source bin/activate
        $ pip install -U -r requirements.txt
        $ pip install psycopg2 # only necessary for production installs

Make the config directory and copy the config file over:

        $ mkdir -p /var/www/.config/streetcrm/
        $ cp config.example.ini /var/www/.config/streetcrm/config.ini

(If you're somewhere other than `/var/www/streetcrm/`, for example
under your home directory because you're doing a development
deployment rather than a production deployment, then you might put the
config file in a different location.  If so, just set the
`STREETCRM_CONFIG` environment variable accordingly, e.g., by running

        $ STREETCRM_CONFIG=~/.config/streetcrm/config.ini; export
STREETCRM_CONFIG

Make sure to specify the path absolutely, not just relative to the
current directory.  If later on you get errors about StreetCRM being
unable to find its config file, failure to set that environment
variable is probably why.)

Wherever you put the `config.ini` file, make sure to change its access
permissions to be readable only by the user the application will run
as, for example:

        $ chmod go-rwx /var/www/.config/streetcrm/config.ini

otherwise you will see a warning every time you launch the app.

Now edit the `config.ini` file and modify the contents to suit your
needs -- at a minimum you probably want something like this:

        [general]
        # The secret key is just a random string of characters that you
        # make up.  It is used by Django for cryptographic signing,
        # session management, etc.  For more information about it, see
        # https://docs.djangoproject.com/en/1.7/ref/settings/#secret-key.
        secret_key = THIS_SHOULD_BE_CHANGED
        # If this is not a production instance, set `debug` to `true`:
        debug = false
        time_zone = "America/Chicago"
        # Set "allowed_hosts" to the domain of the URL people will access
        # the site on (only needed for PostgreSQL, not for SQLite3):
        allowed_hosts = ["example.com"]
        
        # This will be used to construct one of the labels (on
        # Institution), so you may want to use a shorter version of your
        # org name.
        org_name = __YOUR_ORG_HERE__

        [database]
        # This is for PostgreSQL.  But for development purposes, it's
        # fine to use SQLite3 as the database back end.  In that case,
        # say `django.db.backends.sqlite3` here instead:
        engine = django.db.backends.postgresql_psycopg2
        # 'host' is for PostgreSQL; for SQLite3, just comment this out:
        host = localhost
        # 'name' is for PostgreSQL; for SQLite3, say `streetcrm_db`:
        name = streetcrm
        # 'user' is for PostgreSQL; for SQLite3, just comment this out:
        user = streetcrm
        # 'password' is for PostgreSQL; for SQLite3, just comment this out:
        password = DB_PASSWORD_HERE

Now create the tables in the database and setup the initial superuser:

        $ export STREETCRM_CONFIG=/var/www/.config/streetcrm/config.ini
        $ python manage.py migrate auth
        $ python manage.py migrate

If you encounter a "ProgrammingError" or similar error during
migrations, you probably have an old database interfering.  The
solution is just to drop the database (`drop database` in PostgreSQL,
or in SQLite just remove the database file) and run the
`makemigration` and `migrate` steps again.

You've almost certainly changed the organization name in the config
file from `_StreetCRM_Default_Org_`, so you'll need to create a
migration for it to take effect in the label (before you do that, the
label will show "Is this institution a member of StreetCRM Default
Org?").  So, after you've set the org name in the config file, do the
following:

        $ python manage.py makemigrations
        # you'll see something like the following:
        # Migrations for 'streetcrm':
        #  MIGRATION-FILE-NAME.py:
        #    - Alter field is_member on institution
        $ python manage.py migrate
        # Running migrations:
        #  Rendering model states... DONE
        #  Applying streetcrm.MIGRATION-FILE-NAME... OK


If you want to set a different logo for the top left of the screen
(where, by default, you see the StreetCRM logo), from the root of this
repository do the following:

        $ cp /path/to/your-logo-file.png streetcrm/static/images/logo.png

Our logo image is 1029x273px, for reference.  Once your file is there it
will show up in the app.  You can remove it or change it whenever you
like.

To update the main theme color for the StreetCRM instance, change the
`theme_color` setting in `config.ini` (the default in `config.example.ini`
is `"#1e6b27"`).

You might also want to update the colors of the three-line menu icon
that is usually in the upper right of the screen.  The icon image is
`streetcrm/static/images/three_lines_default.png`.  You can change the
colors programmatically like this:

    $ convert three_lines_default.png -fuzz 100% -fill "#YOUR_COLOR" -opaque "#1e6b27" tmp.png
    $ mv tmp.png three_lines_default.png

This is not yet a very good story for how to change the menu colors,
since the result is that a versioned file is modified.  We'll fix that
soon, but in the meantime, the above will at least let you control
your menu icon's color.

Load sample data if necessary
-----------------------------

Optionally, you may load sample data too (but see the warning below):

        $ python manage.py loaddata sample_data.json

The sample data is located in `streetcrm/fixtures/sample_data.json`,
but Django knows where to find it if you just say `sample_data.json`.

If you're loading sample data, you probably need to create a superuser
account too (it's something you should do the first time you
initialize or re-initialize the database).  Use a username that
represents you individually, such as "jrandom", not a generic role
name like "admin".  In the context of StreetCRM, a "superuser" is just
a user who has permission to do anything, and there can be multiple
such users -- you just happen to be creating the first one.

        $ python manage.py createsuperuser

NOTE: The sample data should have been created with this command:

        python manage.py dumpdata --natural-foreign --indent=4       \
                                  -e sessions -e admin               \
                                  -e contenttypes -e auth.Permission \
              > streetcrm/fixtures/sample_data.json

as per
[these](http://stackoverflow.com/questions/27499030/integrity-error-when-loading-fixtures-for-selenium-testing-in-django)
[two](http://stackoverflow.com/questions/853796/problems-with-contenttypes-when-loading-a-fixture-in-django)
StackOverflow questions.  Otherwise, people may get ""UNIQUE
constraint failed" errors when trying to load the data.

Running StreetCRM (developer installation)
------------------------------------------

To run StreetCRM just for testing and development purposes, we
recommend you use Python's built-in web server:

        $ python manage.py runserver

Note that this assumes you are still in the `virtualenv` environment.
If you're not, you'll need to set that up again, which is done with
these two commands seen earlier:

        $ virtualenv --python=python3.4 .
        $ source ./bin/activate

Now you can invoke `python manage.py runserver` from the proper
environment.

In your terminal, you'll see something like

        $ python manage.py runserver
        # Performing system checks...
        #
        # System check identified no issues (0 silenced).
        # January 22, 2017 - 16:36:12
        # Django version 1.8.17, using settings 'streetcrm.settings'
        # Starting development server at http://127.0.0.1:8000/
        # Quit the server with CONTROL-C.

In your browser, visit the URL the terminal messages mention, e.g.,
127.0.0.1:8000/ . You'll see the StreetCRM login screen. In order to
log in, you'll need to create a user; see the superuser creation
instructions in the sample data loading instructions above.

To stop the server, use *Control-C* in the terminal where it's
running.

If you want to exit the `virtualenv` environment, run:

        $ deactivate

To run StreetCRM for production use, see the "HTTP Server" section below.

HTTP Server (production installation)
-------------------------------------

Copy the HTTPD configuration file from the repository to wherever your HTTPD
configuration lives.  We'll assume Debian-standard locations here:

        $ cp /var/www/streetcrm/apache-24.conf /etc/apache2/sites-available/streetcrm.conf

Then edit the file you've just copied with your favorite editor:

        $ favorite-editor /etc/apache2/sites-available/streetcrm.conf

Edit these lines:

        ServerName      - This should be the URL of the site
        ServerAdmin     - This should be the email to contact you on

That should be all you need to edit.  Save the file and exit, then
enable the config you just created:

        $ a2ensite streetcrm

Verify the config is still valid:

        $ apachectl configtest

Fix any errors or warnings it reports.

Reload the HTTPD config and tell systemd to start HTTPD on boot:

        $ systemctl restart apache2
        $ systemctl enable apache2

Updating StreetCRM
-----------------

When updating to a newer version of StreetCRM, run:

        $ pip install -r requirements.txt -U
        $ python manage.py migrate


Old Versions (pre-rename)
------------------------

For instances that were set up with the original name of StreetCRM,
SWOPtact, you may need to do some careful migration management.  To
update to the renamed app, do the following:

        $ cp streetcrm/migrations/0061_rename_tables.py.tmpl streetcrm/migrations/0061_rename_tables.py 
        
        # Now you'll have an unmigrated migration.  This is left as a
        # ".tmpl" file in the repo because new instances shouldn't run
        # it.  It's only necessary for instances that need to manually
        # do the rename.  If you're doing the rename, run the migration
        # as usual.

        $ python manage.py migrate

        # For future migrations (AFTER 61), you'll need to fake up to
        # the previous migration.  That will make future migrations work
        # so that changes introduced in commit 7b4f817 won't cause
        # trouble.

        $ python manage.py migrate --fake streetcrm 0060

        # Then run the next migration, which is likely a name change in the config:
        $ python manage.py migrate streetcrm 0061_auto_20160420_1340

You only need to do the `--fake` command for the first migration
post-rename.  After that, migrations should all work as expected and you can just run

        $ python manage.py migrate

as usual.

