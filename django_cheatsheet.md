Ctr+shift+v to preview

# Django CheatSheet by Sikander

## Install virtualenv

1. pip install virtualenv
2. To create virtual environment name 'env'
   > virtualenv env
3. To activate virtual env
   > "env/scripts/activate

## Install Django

1. > pip install django
2. To start a project
   > django-admin startproject `project_name`

## Setup Postgresql and PgAdmin in your system

### Setup Postgresql

1. Download postgresql installer from https://www.enterprisedb.com/downloads/postgres-postgresql-downloads
2. Run the installer
3. Set the password for db super user (postgres) -> "postgres@27"
4. Just keep clicking next and then click on install.
5. Once the installation is finished, uncheck the stackbuilder option and click on finish.

### Setup PgAdmin

1. Download the pgAdmin installer from https://www.postgresql.org/ftp/pgadmin/pgadmin4/v4.27/windows/
2. Run the installer.
3. Keep the default settings and click on install.
4. Open pgAdmin and create a new database. e.g. `Wd_platform`

## Setup database

1. By default django uses sqlite.
2. To change db setting go to settings.py and change the db setting.

- ` 'ENGINE': 'django.db.backends.postgresql', 'NAME': 'Wd_platform', 'USER': 'postgres', 'PASSWORD': 'postgres@27' 'HOST': 'localhost'`

3. Install psycopg2 to connect django with postgresql
   > pip install psycopg2

## Run the initial migration

> python manage.py migrate

## Create SuperUser

> python manage.py createsuperuser

## Run the server

> python manage.py runserver

## Create app

1. > python manage.py startapp `app_name`
2. add your app to installed_apps in settings.
3. create `templates>app_name>register.html` -> all your templates will be here.

## Install crispy forms

1. > pip install django-crispy-forms
2. Add `crispy_forms` to installed apps in settings.py
3. Set crispy form template
   > CRISPY_TEMPLATE_PACK = 'bootstrap4'
4. To add crispy forms to html template
   > {% load crispy_forms_tags %} > {{form | crispy}}

## Change Login Redirect url

1. > LOGIN_REDIRECT_URL = 'blog-home'
2. > LOGIN_URL = 'login'

### Important tips

1. Always start your django proect with custom user model.
2.

# Heroku setup

1. Download heroku CLI
2. Login to heroku
   > heroku login
3. Install Gunicorn
   > pip install gunicorn
4. Create requirements.txt file
   > pip freeze > requirements.txt
5. Initialize git
   > git init
   > create .gitignore
   > git add -A
   > git status
   > git commit -m 'initial'
6. Create heroku app
   > heroku create `app-name`
7. Push code to heroku
   > git push heroku master
8. create Procfile in root directory
9. install postgres following the guide https://devcenter.heroku.com/articles/heroku-postgresql#local-setup
10. Install postgresql and create databe
    > heroku addons:create heroku-postgresql:hobby-dev
    > use `heroku:pg` to check the available db
11. Install django heroku
    > pip install django-heroku
12. Update the settings file

- import django_heroku
- add `django_heroku.settings(locals())` at the bottom

13. Migrate the db

- ### Two ways to do that
  1.  > heroku run python manage.py runserver
  2.  > heroku run bash

14. Create super user `using bash`
    > python manage.py createsuperuser
