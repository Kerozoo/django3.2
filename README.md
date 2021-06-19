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

# Create `polls` functions

  - Add urls for actions to polls/urls.py

    ```python
    from django.urls import path

    from . import views

    app_name = 'polls'
    urlpatterns = [
        # ex: /polls/
        path('', views.index, name='index'),
        # ex: /polls/5/
        path('<int:question_id>/', views.detail, name='detail'),
        # ex: /polls/5/results/
        path('<int:question_id>/results/', views.results, name='results'),
        # ex: /polls/5/vote/
        path('<int:question_id>/vote/', views.vote, name='vote'),
    ]
    ```

- ## Add index
    
  - Add index action to polls/views.py
        
    - Using `render` example

      ```python
      from django.shortcuts import render

      from .models import Question


      def index(request):
          latest_question_list = Question.objects.order_by('-pub_date')[:5]
          context = {'latest_question_list': latest_question_list}
          return render(request, 'polls/index.html', context)
      ```

    - Using `HttpResponse` and `loader` example

      ```python
      from django.http import HttpResponse
      from django.template import loader

      from .models import Question

      def index(request):
          latest_question_list = Question.objects.order_by('-pub_date')[:5]
          template = loader.get_template('polls/index.html')
          context = {'latest_question_list': latest_question_list}
          return HttpResponse(template.render(context, request))
      ```

  - Add index templates to `polls/templates/polls/index.html` (Create directories)

    ```
    {% if latest_question_list %}
    <ul>
        {% for question in latest_question_list %}
        <li><a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a></li>
        {% endfor %}
    </ul>
    {% else %}
        <p>No polls are available.</p>
    {% endif %}
    ```

  - Access local host

    http://localhost:8080/polls/

- ## Add detail
    
  - Add detail action with 404 to polls/views.py

    - Using `get_object_or_404` example

      ```python
      from django.shortcuts import get_object_or_404, render

      from .models import Question


      def detail(request, question_id):
          question = get_object_or_404(Question, pk=question_id)
          return render(request, 'polls/detail.html', {'question': question})
      ```
        
    - Using `Http404` example

      ```python
      from django.http import Http404
      from django.shortcuts import render

      from .models import Question


      def detail(request, question_id):
          try:
              question = Question.objects.get(pk=question_id)
          except Question.DoesNotExist:
              raise Http404("Question does not exist")
          return render(request, 'polls/detail.html', {'question': question})
      ```

  - Add vote mock action to polls/views.py for not to output AttributeError

    ```python
    from django.http import HttpResponse

    def vote(request, question_id):
        return HttpResponse("You're voting on question %s." % question_id)
    ```

  - Add detail templates to `polls/templates/polls/detail.html` (Create file)

    ```
    <form action="{% url 'polls:vote' question.id %}" method="post">
        {% csrf_token %}
        <fieldset>
            <legend><h1>{{ question.question_text }}</h1></legend>
            {% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}
            {% for choice in question.choice_set.all %}
                <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
                <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
            {% endfor %}
        </fieldset>
        <input type="submit" value="Vote">
    </form>
    ```

  - Access local host

    - http://localhost:8080/polls/1/
    - http://localhost:8080/polls/99999/

- ## Add vote and results

  - Add vote function to polls/views.py

    ```python
    from django.http import HttpResponseRedirect
    from django.shortcuts import get_object_or_404, render
    from django.urls import reverse

    from .models import Choice, Question

    def vote(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        try:
            selected_choice = question.choice_set.get(pk=request.POST['choice'])
        except (KeyError, Choice.DoesNotExist):
            # Redisplay the question voting form.
            return render(request, 'polls/detail.html', {
                'question': question,
                'error_message': "You didn't select a choice.",
            })
        else:
            selected_choice.votes += 1
            selected_choice.save()
            # Always return an HttpResponseRedirect after successfully dealing
            # with POST data. This prevents data from being posted twice if a
            # user hits the Back button.
            return HttpResponseRedirect(reverse('polls:results', args=(question.id,)))
    ```

  - Add results action to polls/views.py

    ```python
    from django.shortcuts import render, get_object_or_404

    def results(request, question_id):
        question = get_object_or_404(Question, pk=question_id)
        return render(request, 'polls/results.html', {'question': question})
    ```

  - Add results templates to `polls/templates/polls/results.html` (Create file)

    ```
    <h1>{{ question.question_text }}</h1>

    <ul>
        {% for choice in question.choice_set.all %}
        <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
        {% endfor %}
    </ul>

    <a href="{% url 'polls:detail' question.id %}">Vote again?</a>
    ```

  - Access local host then `vote`

    - http://localhost:8080/polls/1/

# Testing `polls` model

- ## Add `was_published_recently()` to Question model with bug

  ```python
  from django.db import models
  from django.utils import timezone
  import datetime

  class Question(models.Model):
      #…
      def was_published_recently(self):
          # This method returns True even on future dates (bug)
          return self.pub_date >= timezone.now() - datetime.timedelta(days=1)
  ```

- ## Add test of `was_published_recently()` to polls/tests.py

  - Add test

    ```python
    import datetime

    from django.test import TestCase
    from django.utils import timezone

    from .models import Question


    class QuestionModelTests(TestCase):

        def test_was_published_recently_with_future_question(self):
            """
            was_published_recently() returns False for questions whose pub_date
            is in the future.
            """
            time = timezone.now() + datetime.timedelta(days=30)
            future_question = Question(pub_date=time)
            self.assertIs(future_question.was_published_recently(), False)
    ```

  - Check the test is FAILED

    - Exec test

      ```
      docker-compose exec app python app/manage.py test polls
      ```

    - Check the result

      ```
      Creating test database for alias 'default'...
      System check identified no issues (0 silenced).
      F
      ======================================================================
      FAIL: test_was_published_recently_with_future_question (polls.tests.QuestionModelTests)
      was_published_recently() returns False for questions whose pub_date
      ----------------------------------------------------------------------
      Traceback (most recent call last):
        File "/code/app/polls/tests.py", line 18, in test_was_published_recently_with_future_question
          self.assertIs(future_question.was_published_recently(), False)
      AssertionError: True is not False

      ----------------------------------------------------------------------
      Ran 1 test in 0.007s

      FAILED (failures=1)
      Destroying test database for alias 'default'...
      ```

- ## Fix bug of `was_published_recently()`

  ```python
  def was_published_recently(self):
      now = timezone.now()
      return now - datetime.timedelta(days=1) <= self.pub_date <= now
  ```

  - Check the test is OK

    - Exec test

      ```
      docker-compose exec app python app/manage.py test polls
      ```

    - Check the result

      ```
      Creating test database for alias 'default'...
      System check identified no issues (0 silenced).
      .
      ----------------------------------------------------------------------
      Ran 1 test in 0.002s

      OK
      Destroying test database for alias 'default'...
      ```
