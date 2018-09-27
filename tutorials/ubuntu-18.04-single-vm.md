# MyTardis 4.0 on Ubuntu 18.04 LTS
This guide assumes that you already have an Ubuntu 18.04 LTS server or virtual machine (VM) running. It's also assumes that you have some familiarity setting up a local development enviroment for MyTardis following [this guide](https://mytardis.readthedocs.io/en/develop/admin/install.html).

## Getting started
We need to install some dependencies. From [mytardis.readthedocs.io](https://mytardis.readthedocs.io/en/develop/admin/install.html):
```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get update
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install git libldap2-dev libmagickwand-dev libsasl2-dev \
   libssl-dev libxml2-dev libxslt1-dev libmagic-dev curl gnupg \
   python-dev python-pip python-virtualenv virtualenvwrapper \
   zlib1g-dev libfreetype6-dev libjpeg-dev

ubuntu@mytardis-ubuntu18:~$ curl -sL https://deb.nodesource.com/setup_10.x | sudo -E bash -
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install -y nodejs
```

## Installing MyTardis
Let’s create a group and user for our web app:
```
ubuntu@mytardis-ubuntu18:~$ sudo groupadd -r mytardis
ubuntu@mytardis-ubuntu18:~$ sudo useradd -m -r -g mytardis -s /bin/bash mytardis
```

Let’s switch to the new user:
```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
```

Clone mytardis:
```
mytardis@mytardis-ubuntu18:~$ git clone https://github.com/mytardis/mytardis.git
mytardis@mytardis-ubuntu18:~$ cd mytardis/
```

Create and activate virtualenv:
```
mytardis@mytardis-ubuntu18:~/mytardis$ source /etc/bash_completion.d/virtualenvwrapper
mytardis@mytardis-ubuntu18:~/mytardis$ mkvirtualenv mytardis
Running virtualenv with interpreter /usr/bin/python2
New python executable in /home/mytardis/.virtualenvs/mytardis/bin/python2
Also creating executable in /home/mytardis/.virtualenvs/mytardis/bin/python
Installing setuptools, pkg_resources, pip, wheel...done.
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ 
```

Install pip dependencies:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ pip install -U -r requirements.txt
```

Install pip dependencies for Postgresql database engine:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ pip install -U -r requirements-postgres.txt
```

Install Javascript dependencies:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ npm install
```

Make a directory to store the uploaded files:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ mkdir -p var/store
```
Note: In production, we would most likely mount a CIFS or NFS volume to host the data.

Let’s create a secret key:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python -c "import os; from random import choice; key_line = '%sSECRET_KEY=\"%s\"  # generated from build.sh\n' % ('from tardis.settings_changeme import * \n\n' if not os.path.isfile('tardis/settings.py') else '', ''.join([choice('abcdefghijklmnopqrstuvwxyz0123456789@#%^&*(-_=+)') for i in range(50)])); f=open('tardis/settings.py', 'a+'); f.write(key_line); f.close()"
```
and run the mytardis unittests to see if every seems okay with the environment:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python test.py
nosetests --verbosity=1
Creating test database for alias 'default'...

....................................................................................................................S.S.S.S.S.S.S.........................................................S.S.S...S..................
----------------------------------------------------------------------
Ran 202 tests in 49.521s

OK (SKIP=11)
Destroying test database for alias 'default'...
```

Now let's run mytardis's Javascript unit tests.  This requires running the full "npm install" as above, rather than just running "npm install --production" to install the bare minimum Javascript dependencies.

```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ npm test

...
>> 5 tests completed with 0 failed, 0 skipped, and 0 todo. 
>> 22 assertions (in 120ms), passed: 22, failed: 0

Done.
```

Okay for now we are just going to use the `sqlite` database for testing. In fact, this is already configured in MyTardis’s  default settings [here](https://github.com/mytardis/mytardis/blob/develop/tardis/default_settings/database.py). Note we’ll need to do this again when we switch to a production database.

Let's run the migrations to create the database tables in our SQLite database:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py migrate
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createcachetable default_cache
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createcachetable celery_lock_cache
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ ls -lh db.sqlite3 
-rw-r--r-- 1 mytardis mytardis 604K Sep 25 03:10 db.sqlite3
```

Finally, we collect our static assets:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py collectstatic

527 static files copied to '/home/mytardis/mytardis/static', 572 post-processed.
```

While we’re here, let’s also create a superuser for our admin:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createsuperuser
```

Let's check if we can run MyTardis using Django's built-in web server (not recommended for production):
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ ./manage.py runserver
```

And from another Terminal:
```
ubuntu@mytardis-ubuntu18:~$ wget http://127.0.0.1:8000
--2018-09-25 03:21:23--  http://127.0.0.1:8000/
Connecting to 127.0.0.1:8000... connected.
HTTP request sent, awaiting response... 200 OK
Length: 10605 (10K) [text/html]
Saving to: 'index.html’

index.html          100%[===================>]  10.36K  --.-KB/s    in 0s      

2018-09-25 03:21:23 (53.2 MB/s) - 'index.html’ saved [10605/10605]
```
Okay looking good.

At this point we could have a play with MyTardis on a local machine, but there are limitations in SQLite (i.e., concurrent DB access) that prevent it from being able to power a full MyTardis. We’ll come back to this later.

## Switching to Postgres DB
To run in production, we really need to switch databases.  Let’s install postgres:
```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install postgresql postgresql-contrib
```
Note: we have switched back to the `ubuntu` user.

Let’s create a database and credentials for `mytardis`
```
ubuntu@mytardis-ubuntu18:~$ sudo -u postgres psql
psql (10.5 (Ubuntu 10.5-0ubuntu0.18.04))
Type "help" for help.

postgres=# CREATE DATABASE mytardis_db;
postgres=# CREATE USER mytardis WITH PASSWORD 'makeitstronger';
postgres=# ALTER ROLE mytardis SET client_encoding TO 'utf8';
postgres=# ALTER ROLE mytardis SET default_transaction_isolation TO 'read committed';
postgres=# ALTER ROLE mytardis SET timezone TO 'UTC';
postgres=# \q
```
The following guide from DigitalOcean: [How To Set Up Django with Postgres, Nginx, and Gunicorn on Ubuntu 18.04](https://www.digitalocean.com/community/tutorials/how-to-set-up-django-with-postgres-nginx-and-gunicorn-on-ubuntu-18-04) provides more detailed information about installing and configuring Postgresql on Ubuntu 18.04.

Also note Django's recommendations for setting the timezone to `UTC` for the database user when `USE_TZ` is True in Django's settings: https://docs.djangoproject.com/en/1.11/ref/databases/#optimizing-postgresql-s-configuration

Let’s switch back to our `mytardis` user and configure the database in Django:
```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ source /etc/bash_completion.d/virtualenvwrapper
mytardis@mytardis-ubuntu18:~$ workon mytardis
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ vi tardis/settings.py
```

Add the following to  `tardis/settings.py` :
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'mytardis_db',
        'USER': 'mytardis',
        'PASSWORD': 'makeitstronger',
        'HOST': 'localhost',
        'PORT': '',
    }
}
```

We now need to re-run the migrations first and then create the `default_cache` and `celery_lock_cache` tables:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py migrate
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createcachetable default_cache
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ python manage.py createcachetable celery_lock_cache
```

## Serving through Nginx

Django provides a demo web server (which can be run with `python manage.py runserver`), however it should not be used in production, as advised by the Django docs at https://docs.djangoproject.com/en/1.11/ref/django-admin/:

> DO NOT USE THIS SERVER IN A PRODUCTION SETTING. It has not gone through security audits
> or performance tests. (And that’s how it’s gonna stay. We’re in the business of making
> Web frameworks, not Web servers, so improving this server to be able to handle a
> production environment is outside the scope of Django.)

As the `ubuntu` user, let's install nginx:
```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install nginx
```

Let’s create a simple Nginx config that proxies requests from port 80 (HTTP) to http://localhost:8000/ (i.e. MyTardis running with Django's `manage.py runserver`).
```
ubuntu@mytardis-ubuntu18:~$ sudo rm /etc/nginx/sites-enabled/default
ubuntu@mytardis-ubuntu18:~$ sudo vi /etc/nginx/sites-available/mytardis
```

Add the following to `/etc/nginx/sites-available/mytardis`:
```
server {
  listen 80;
  server_name 118.138.234.127;  # Replace this with your server's IP address

  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
    root /home/mytardis/mytardis;
  }

  location / {
    include proxy_params;

    # We'll use gunicorn in production:
    # proxy_pass http://unix:/tmp/gunicorn.socket;

    # But first we'll test proxying nginx to Django's manage.py runserver:
    proxy_pass http://127.0.0.1:8000;    
  }
}
```

Now link this config into `/etc/nginx/site-enabled`:
```
ubuntu@mytardis-ubuntu18:~$ sudo ln -s /etc/nginx/sites-available/mytardis /etc/nginx/sites-enabled/
```

Test nginx config:
```
ubuntu@mytardis-ubuntu18:~$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

If you get no errors, restart NGINX.
```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl restart nginx
```

By default Ubuntu iptables rules don’t allow port 80 through the firewall. We can use `ufw`  to control iptables and we can use the ‘Nginx Full’ profile to allow this:
```
ubuntu@mytardis-ubuntu18:~$ sudo ufw enable
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
ubuntu@mytardis-ubuntu18:~$ sudo ufw allow OpenSSH
ubuntu@mytardis-ubuntu18:~$ sudo ufw allow 'Nginx Full'
ubuntu@mytardis-ubuntu18:~$ sudo ufw status
Status: active

To                         Action      From
--                         ------      ----
OpenSSH                    ALLOW       Anywhere                  
Nginx Full                 ALLOW       Anywhere                  
OpenSSH (v6)               ALLOW       Anywhere (v6)             
Nginx Full (v6)            ALLOW       Anywhere (v6)
```

Note: it is **important** to allow OpenSSH too otherwise you will not be able to SSH back your VM later.
Note: you could iptables rules directly.
Note: If you are using a cloud user, you may also have security groups enabled. Make sure you allow port 80 (i.e., HTTP) in your security groups too.

If we run our MyTardis server again now, it should be accessible by visiting the IP address of your server in the browser.
```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ source ~/.virtualenvs/mytardis/bin/activate
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ ./manage.py runserver
```

Navigate to http://serverip/

An important thing to note is that this is only connecting over HTTP, so the connection is not secure.

## Setting up a gunicorn service

As Django's `manage.py runserver` is not suitable for production, Django applications are typically deployed with WSGI as described at https://docs.djangoproject.com/en/1.11/howto/deployment/wsgi/

MyTardis deployments typically use the Gunicorn WSGI server.

Instead of `./mytardis.py runserver` we can run via gunicorn using the gunicorn settings in the mytardis repo:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ gunicorn -c gunicorn_settings.py \
  -b unix:/tmp/gunicorn.socket -b 127.0.0.1:8000 --log-syslog  wsgi:application
```
You can press Control-C to stop the Gunicorn processes - normally they would be launched from a Systemd service, not from an interactive Terminal session.

Make a note of where the `gunicorn` bin is in you virtualenv:
```
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ which gunicorn
/home/mytardis/.virtualenvs/mytardis/bin/gunicorn
```

Let's create a Systemd service to manage running gunicorn. Add the following to `/etc/systemd/system/gunicorn.service`:
```
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/gunicorn -c gunicorn_settings.py -b unix:/tmp/gunicorn.socket -b 127.0.0.1:8000 --log-syslog wsgi:application

[Install]
WantedBy=multi-user.target
```

Note that ExecStart is basically calling gunicorn from our virtualenv (see above) with the previous command we used to start it manually.

We can now launch the service and check the status:
```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl start gunicorn
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status gunicorn
● gunicorn.service - gunicorn daemon
   Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-09-25 04:02:41 UTC; 4s ago
```

We should now update our nginx config to proxy HTTP requests to Gunicorn's socket (/tmp/gunicorn.socket)
instead of to the Django manage.py runserver at http://127.0.0.1:8000

Our /etc/nginx/sites-available/mytardis should be updated to look like this:
```
server {
  listen 80;
  server_name 118.138.234.127;  # Replace this with your server's IP address

  location = /favicon.ico { access_log off; log_not_found off; }
  location /static/ {
    root /home/mytardis/mytardis;
  }

  location / {
    include proxy_params;
    proxy_pass http://unix:/tmp/gunicorn.socket;
  }
}
```
We can reload the configuration with:
```
ubuntu@mytardis-ubuntu18:~$ sudo service nginx reload
```

Now let's check again that we can access MyTardis through the browser via http://serverip/

You can also check that your requests are being logged in MyTardis's request.log:

```
ubuntu@mytardis-ubuntu18:~$ tail /home/mytardis/mytardis/request.log
```

MyTardis's default log levels can be found in `tardis/default_settings/logging.py`

## Scheduled and asynchronous tasks using Celery
MyTardis has several tasks that operate asynchronously or on a schedule. This is done using [Celery](http://www.celeryproject.org). Celery uses a messaging queue and while there are some default configs in for MyTardis to use the database as the broker, this is not very efficient. A much better method is to use RabbitMQ as the broker. Let’s install RabbitMQ and configure it:

As the `ubuntu` user:
```
ubuntu@mytardis-ubuntu18:~$ sudo apt-get install rabbitmq-server

```
Note: this installs from the Ubuntu repos. In general, it's better to install the most up-to-date RabbitMQ. See [this page](https://www.rabbitmq.com/install-debian.html)


We need to do a bit of setup to use RabbitMQ in Celery. This includes adding a user and a virtual host:
```
sudo rabbitmqctl add_user mytardis makeitreallystrong
sudo rabbitmqctl add_vhost mytardisvhost
sudo rabbitmqctl set_user_tags mytardis mytardistag
sudo rabbitmqctl set_permissions -p mytardisvhost mytardis ".*" ".*" ".*"
```
Note: this information was sourced from [Using RabbitMQ — Celery 4.1.0 documentation](http://docs.celeryproject.org/en/latest/getting-started/brokers/rabbitmq.html)

Now we need to add the following to `/home/mytardis/mytardis/tardis/settings.py`:
```
CELERY_RESULT_BACKEND = 'amqp'
BROKER_URL = 'amqp://mytardis:makeitreallystrong@localhost:5672/mytardisvhost'
```

Let’s test whether this works from within our mytardis virtualenv:
```
ubuntu@mytardis-ubuntu18:~$ sudo -i -u mytardis
mytardis@mytardis-ubuntu18:~$ source ~/.virtualenvs/mytardis/bin/activate
(mytardis) mytardis@mytardis-ubuntu18:~$ cd mytardis
(mytardis) mytardis@mytardis-ubuntu18:~/mytardis$ DJANGO_SETTINGS_MODULE=tardis.settings celery worker -A tardis.celery.tardis_app
```

You can press Control-C to terminate your temporary celery worker instance.

Like gunicorn, the celery worker processes should be run as a Systemd service.

Add the following to `/etc/systemd/system/celeryworker.service`:
```
[Unit]
Description=celeryworker daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
Environment=DJANGO_SETTINGS_MODULE=tardis.settings
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/celery worker \
  -A tardis.celery.tardis_app \
  -c 2 -Q celery,default -n "allqueues.%%h"

[Install]
WantedBy=multi-user.target
```

We also need a celery beat Systemd service for scheduling tasks.

Add the following to `/etc/systemd/system/celerybeat.service`:
```
[Unit]
Description=celerybeat daemon
After=network.target

[Service]
User=mytardis
Group=mytardis
WorkingDirectory=/home/mytardis/mytardis
Environment=DJANGO_SETTINGS_MODULE=tardis.settings
ExecStart=/home/mytardis/.virtualenvs/mytardis/bin/celery beat \
  -A tardis.celery.tardis_app --loglevel INFO

[Install]
WantedBy=multi-user.target
```

It's good practice in MyTardis to have a Celery beat task that makes sure any files that missed verification get reverified. We can add this to `tardis/settings.py`:
```
from datetime import timedelta
from .celery import tardis_app
tardis_app.conf.CELERYBEAT_SCHEDULE = {
    'task': "tardis_portal.verify_dfos",
    'schedule': timedelta(seconds=3600)
}
```

We can now start the celeryworker and celerybeat services:
```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl start celeryworker
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status celeryworker
● celeryworker.service - celeryworker daemon
   Loaded: loaded (/etc/systemd/system/celeryworker.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-09-25 04:51:39 UTC; 12s ago
```

```
ubuntu@mytardis-ubuntu18:~$ sudo systemctl start celerybeat
ubuntu@mytardis-ubuntu18:~$ sudo systemctl status celerybeat
● celerybeat.service - celerybeat daemon
   Loaded: loaded (/etc/systemd/system/celerybeat.service; disabled; vendor preset: enabled)
   Active: active (running) since Tue 2018-09-25 05:44:19 UTC; 3s ago
```

## That's it
We should now have a fully functional basic version of MyTardis.

### TODO

- SFTP configuration
- Working with external storage
- Adding filters