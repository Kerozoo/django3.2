# Starting django project

- Commentout `command` in docker-compose.yml (if no app in apps)

  ```diff
  - command: python3 app/manage.py runserver 0.0.0.0:8000
  # command: python3 app/manage.py runserver 0.0.0.0:8000
  ```

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

# Activate `polls` Model and add classes

  - Activate `polls` app in app project (app/app/settings.py)

    ```diff
    INSTALLED_APPS = [
    +   'polls.apps.PollsConfig',
        'django.contrib.admin',
        'django.contrib.auth',
        'django.contrib.contenttypes',
        'django.contrib.sessions',
        'django.contrib.messages',
        'django.contrib.staticfiles',
    ]
    ```

  - Add model classes to polls/models.py

    ```
    from django.db import models


    class Question(models.Model):
        question_text = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')


    class Choice(models.Model):
        question = models.ForeignKey(Question, on_delete=models.CASCADE)
        choice_text = models.CharField(max_length=200)
        votes = models.IntegerField(default=0)
    ```

  - Add `__str__` to polls/models.py

    ```
    class Question(models.Model):
        …

        def __str__(self):
            return self.question_text

    class Choice(models.Model):
        …

        def __str__(self):
            return self.choice_text
    ```

# Migrate polls models

  - Makemigration

    - Command

      ```
      docker-compose exec app python app/manage.py makemigrations polls
      ```

    - Result

      ```
      Migrations for 'polls':
        app/polls/migrations/0001_initial.py
          - Create model Question
          - Create model Choice
      ```

  - Check migration as sql

    - Command

      ```
      docker-compose exec app python app/manage.py sqlmigrate polls 0001
      ```

    - Result (MySQL)

      ```
      --
      -- Create model Question
      --
      CREATE TABLE `polls_question` (
        `id` bigint AUTO_INCREMENT NOT NULL PRIMARY KEY,
        `question_text` varchar(200) NOT NULL,
        `pub_date` datetime(6) NOT NULL
      );
      --
      -- Create model Choice
      --
      CREATE TABLE `polls_choice` (
        `id` bigint AUTO_INCREMENT NOT NULL PRIMARY KEY,
        `choice_text` varchar(200) NOT NULL,
        `votes` integer NOT NULL,
        `question_id` bigint NOT NULL
      );
      ALTER TABLE `polls_choice`
        ADD CONSTRAINT `polls_choice_question_id_c5b4b260_fk_polls_question_id`
          FOREIGN KEY (`question_id`)
          REFERENCES `polls_question` (`id`);
      ```

  - Migrate migrations

    - Command

      ```
      docker-compose exec app python app/manage.py migrate
      ```

    - Result

      ```
      Operations to perform:
        Apply all migrations: admin, auth, contenttypes, polls, sessions
      Running migrations:
        Applying polls.0001_initial... OK
      ```

# Test polls model in django shell

  - Reload container
      
    ```
    docker-compose down
    docker-compose up -d
    ```

  - Exec django shell

    ```
    docker-compose exec app python app/manage.py shell
    ```

  - Add data to `polls` model

    ```
    >>> from polls.models import Choice, Question
    >>> from django.utils import timezone
      
    # Create Question
    >>> question = Question(question_text="test1", pub_date=timezone.now())
    >>> question.save()
    >>> question.id
    1

    # Create related choices
    >>> question.choice_set.create(choice_text='choice_test_1', votes=0)
    <Choice: choice_test_1>
    >>> question.choice_set.create(choice_text='choice_test_2', votes=0)
    <Choice: choice_test_2>
    >>> question.choice_set.create(choice_text='choice_test_3', votes=0)
    <Choice: choice_test_3>
    ```

  - Check data

    ```
    >>> from polls.models import Choice, Question

    # Check question
    >>> question = Question.objects.get(pk=1)
    >>> question
    <Question: test1>

    # Check related choices
    >>> choices = question.choice_set.all()
    >>> choices
    <QuerySet [<Choice: choice_test_1>, <Choice: choice_test_2>, <Choice: choice_test_3>]>
    >>> choices[0]
    <Choice: choice_test_1>
    >>> choices[1]
    <Choice: choice_test_2>
    >>> choices[2]
    <Choice: choice_test_3>
    >>> question.choice_set.count()
    3
    ```
