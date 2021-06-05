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

# Add `polls` application and urls

  - Add `polls` application

    ```
    docker-compose exec app python app/manage.py startapp polls
    ```

  - Move apps/polls to apps/app/polls

    ```diff
    apps/
      app/
        ├ app/
    +   ├ polls/
        ├ manage.py
    - polls/
      requirements.txt
    ```

  - Add polls's url

    - polls/views.py

      ```
      from django.http import HttpResponse


      def index(request):
          return HttpResponse("Hello, world. You're at the polls index.")
      ```

    - polls/urls.py(create file)

      ```
      from django.urls import path
      from . import views


      urlpatterns = [
          path('', views.index, name='index'),
      ]
      ```

    - app/app/urls.py

      ```diff
      - from django.urls import path

      -urlpatterns = [
      -    path('admin/', admin.site.urls),
      -]

      from django.urls import include, path

      urlpatterns = [
          path('polls/', include('polls.urls')),
          path('admin/', admin.site.urls),
      ]
      ```

  - Restart container then access to polls app

    ```
    docker-compose down
    docker-compose up
    ```

    http://localhost:8080/polls