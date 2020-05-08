---
layout: post
title: "Alembic migrations autogeneration"
date: 2020-05-08 17:00:00 +0300
categories: Python
---

When it comes to autogeneration database migration files, I like the Django approach. Django compares migrations with model definitions and generates migration files, it is well described in [this](https://realpython.com/digging-deeper-into-migrations/#how-django-detects-changes-to-your-models) post.
 
Alembic migration autogeneration is based on a comparison of table metadata in the application and actual database state. In other words, you need to have a database with the actual state to detect changes in model definitions. It's not convenient, but there is a workaround that will make it more Django-like.

## Application.

This is our test Flask application with [Flask-Migrate](https://flask-migrate.readthedocs.io) and [Flask-SQLAlchemy](https://flask-sqlalchemy.palletsprojects.com) extensions:
```python
# app.py

import os

from flask import Flask
from flask_migrate import Migrate
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = os.getenv('DB_URL')

db = SQLAlchemy(app)
migrate = Migrate(app, db)


class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80))
```

The application gets a database URL from the environment variables.

Generate a new migrations repository:
```
$ FLASK_APP=app.py flask db init
  Creating directory migrations ...  done
  Creating directory migrations/versions ...  done
  Generating migrations/env.py ...  done
  Generating migrations/alembic.ini ...  done
  Generating migrations/README ...  done
  Generating migrations/script.py.mako ...  done
  Please edit configuration/connection/logging settings in 'migrations/alembic.ini' before proceeding.
```

## Migrations auotogeneration.

The actual workaround is the following:

* Create a temporary database
* Set the database URL in environment variables
* Apply all existent migrations
* Generate new migrations

Let's wrap all these steps in a Makefile:
```
# Makefile

makemigrations:
	find . -name _temp.db -delete
	DB_URL="sqlite:///_temp.db" flask db upgrade
	DB_URL="sqlite:///_temp.db" flask db migrate
	find . -name _temp.db -delete
```

Now you can generate migrations:
```
$ make makemigrations
find . -name _temp.db -delete
DB_URL="sqlite:///_temp.db" flask db upgrade
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
DB_URL="sqlite:///_temp.db" flask db migrate
INFO  [alembic.runtime.migration] Context impl SQLiteImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
INFO  [alembic.autogenerate.compare] Detected added table 'user'
  Generating migrations/versions/eb3206646776_.py ...  done
find . -name _temp.db -delete
```

***

Feel free to contact me on [LinkedIn](https://www.linkedin.com/in/art-vasilyev/) or drop me an [email](mailto:artem.v.vasilyev@gmail.com).
