# Starting django project

- Build and up container

  ```
  docker-compose up
  ```

- Create django project

  ```
  docker-compose exec app django-admin startproject app
  ```

- Setting app db in apps/app/app/settings.py with env

  - Add import getenv

    ```
    from os import getenv
    ```

  - Replace DATABASES

    ```diff
    -DATABASES = {
    -    'default': {
    -      'ENGINE': 'django.db.backends.sqlite3',
    -      'NAME': BASE_DIR / 'db.sqlite3',
    -    }
    -}

    DATABASES = {
        'default': {
            'ENGINE': 'django.db.backends.mysql',
            'NAME': getenv('MYSQL_NAME'),
            'USER': getenv('MYSQL_USER'),
            'PASSWORD': getenv('MYSQL_PASSWORD'),
            'HOST': getenv('MYSQL_HOST'),
            'PORT': 3306,
        }
    }
    ```
  - Migrate

    ```
    docker-compose exec app python app/manage.py migrate
    ```

- Uncomment `command` in docker-compose.yml then restart container

  - Uncomment `command` in docker-compose.yml

    ```diff
    -# command: python3 app/manage.py runserver 0.0.0.0:8000
    command: python3 app/manage.py runserver 0.0.0.0:8000
    ```

  - Restart container

    ```
    docker-compose down
    docker-compose up
    ```

  - Access local host

    http://localhost:8080/