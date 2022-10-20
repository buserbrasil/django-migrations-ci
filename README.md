# django-migrations-ci
Optimizations to run less migrations on CI

Django migrations are slow because of state recreation for every migration and other internal Django magic.

In the past, I tried to optimize that, but discovered it's a [running issue](https://code.djangoproject.com/ticket/29898).

## Assumptions

1. I want to run my migrations on CI. It is a sanity check, even if they are generated by Django in some cases.
1. I don't have to run migrations all the time. If migrations didn't change, I can reuse them.
1. The solutions probably are CI-specific and database-specific, but it is possible to write code for a generic workflow and snippets to support each CI.

## Idea

### Generate database state from migrations
GitLab CI (the one I have more experience, but I'm sure others have similar features) has caching based on versioned files.

I can cache database state, based on migrations files (and installed dependencies?).

When cache exists, load this database state (restore?) before your tests and run your tests without migrations.

When cache doesn't exist, run migrations and dump the final state for an SQL file.

Dump and restore are database-specific, but possible to handle all Django supported databases.


## Workflow

This is how the "run test" CI job should work.

```
restore djangomigrations.sql from CI cache

if djangomigrations.sql exists on cache:
  Restore djangomigrations.sql to test database
  Clone the restored test database to run threaded tests
else:
  Run `migrate` command
  Dump test database to djangomigrations.sql

save djangomigrations.sql to CI cache
```

## Cache example on GitHub

TODO #1, I never did it but I'm sure it is possible in some way. See #1

## Cache example on GitLab

Still have to abstract `psql/pg_dump/pg_restore`, but I expect something like that will work:

```
test_job:
  stage: test
  script:
    - head djangomigrations.sql || echo 'djangomigrations.sql does not exist.'
    - |
      if [ -f djangomigrations.sql ]; then
        psql -h $DB_HOST -U $POSTGRES_USER -c "CREATE DATABASE test_$DB_NAME;"
        pg_restore -h $DB_HOST -U $POSTGRES_USER -d test_$DB_NAME djangomigrations.sql
      else
        ./manage.py setup_test_db
        pg_dump -F c -h $DB_HOST -U $POSTGRES_USER test_$DB_NAME > djangomigrations.sql
      fi
    - ./manage.py clone_test_db
    - pytest -n $(nproc)
  cache:
    key:
      # GitLab docs say it accepts only two files, but for some reason it works with wildcards too.
      # You can't add more than two lines here.
      files:
        - "requirements.txt"
        - "*/migrations/*.py"
    paths:
      - djangomigrations.sql
 ```
