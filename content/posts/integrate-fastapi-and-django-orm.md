+++
title = "The integration of FastAPI and Django ORM"
date = "2020-10-12"
cover = ""
tags = ["Python", "web", "fastapi", "django"]
description = "A step-by-step guide to integrating FastAPI and Django ORM"
showFullContent = false
+++

This is the English translation of the Japanese original post at [qiita](https://qiita.com/kigawas/items/80e48ccce98a35f65fff), with some modifications.

## Motivation

Recently FastAPI is [growing incredibly](https://star-history.t9t.io/#tiangolo/fastapi). It's blazingly fast and painless to develop, with [5~10x performance enhancement](https://www.techempower.com/benchmarks/#section=data-r19&hw=ph&test=fortune&l=zijzen-1r) over Django or Flask.

I really want to switch to FastAPI from Django, however, it's not that easy to give up Django and its user system as well as admin page totally. I know it sounds greedy, but in fact there is such convenience. This time I'll show you how to integrate FastAPI and Django ORM simply and quickly.

> There's also a demerit undoubtedly. Django ORM is not asynchronous and this will hurt performance.
>
> If you'd like to improve, you may consider using [orm](https://github.com/encode/orm) or [gino](https://github.com/python-gino/gino) to rewrite some logic.
>
> By the way, Django ORM is also [having a plan](https://docs.djangoproject.com/en/3.1/topics/async/) for supporting asyncio.

## Directory structure

Let's talk about the directory structure first. You can just follow Django's [tutorial](https://docs.djangoproject.com/en/3.1/intro/tutorial01/) to create the scaffold.

```bash
django-admin startproject mysite
django-admin startapp poll
```

If you're done, delete `views.py` and `models.py` and prepare the folders for FastAPI like below.

```bash
$ tree -L 3 -I '__pycache__|venv' -P '*.py'
.
├── manage.py
├── mysite
│   ├── __init__.py
│   ├── asgi.py
│   ├── settings.py
│   ├── urls.py
│   └── wsgi.py
└── polls
    ├── __init__.py
    ├── adapters
    │   └── __init__.py
    ├── admin.py
    ├── apps.py
    ├── migrations
    │   └── __init__.py
    ├── models
    │   └── __init__.py
    ├── routers
    │   └── __init__.py
    ├── schemas
    │   └── __init__.py
    └── tests.py

7 directories, 15 files
```

The usage of each directory:

- `models`: Django ORM
- `routers`: FastAPI routers
- `schemas`: FastAPI Pydantic models
- `adapters`: The adapters to convert Django ORMs to Pydantic validators

## Set up some data

Let's refer to the [Django documentation](https://docs.djangoproject.com/en/3.1/intro/tutorial02/) and insert some data:

```python
>>> from polls.models import Choice, Question
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())
>>> q.save()
```

## FastAPI integration

For simplicity, the codes below are all in the `__init__.py` file of each folder, and I'll also ignore some `import` statements.

### `schemas`

```python
from django.db import models
from pydantic import BaseModel as _BaseModel


class BaseModel(_BaseModel):
    @classmethod
    def from_model(cls, instance: models.Model):
        kwargs = {}
        for k, v in cls.__fields__.items():
            instance_attr = getattr(instance, k)
            if isinstance(instance_attr, models.Model):
                instance_attr = v.type_.from_model(instance_attr)
            kwargs[k] = instance_attr

        return cls(**kwargs)

    @classmethod
    def from_models(cls, instances: List[models.Model]):
        return [cls.from_model(inst) for inst in instances]


class FastQuestion(BaseModel):
    question_text: str
    pub_date: datetime


class FastQuestions(BaseModel):
    items: List[FastQuestion]

    @classmethod
    def from_qs(cls, qs):
        return cls(items=FastQuestion.from_models(qs))


class FastChoice(BaseModel):
    question: FastQuestion
    choice_text: str


class FastChoices(BaseModel):
    items: List[FastChoice]

    @classmethod
    def from_qs(cls, qs):
        return cls(items=FastChoice.from_models(qs))
```

### `adapters`

```python
ModelT = TypeVar("ModelT", bound=models.Model)


def retrieve_object(model_class: Type[ModelT], id: int) -> ModelT:
    instance = model_class.objects.filter(pk=id).first()
    if not instance:
        raise HTTPException(status_code=404, detail="Object not found.")
    return instance


def retrieve_question(
    q_id: int = Path(..., description="get question from db")
) -> Question:
    return retrieve_object(Question, q_id)


def retrieve_choice(c_id: int = Path(..., description="get choice from db")):
    return retrieve_object(Choice, c_id)


def retrieve_questions():
    return Question.objects.all()


def retrieve_choices():
    return Choice.objects.all()
```

### `routers`

```python
router = APIRouter()

@router.get("/")
def get_questions(
    questions: List[Question] = Depends(adapters.retrieve_questions),
) -> FastQuestions:
    return FastQuestions.from_qs(questions)


@router.get("/{q_id}")
def get_question(
    question: Question = Depends(adapters.retrieve_question),
) -> FastQuestion:
    return FastQuestion.from_model(question)

@router.get("/")
def get_choices(
    choices: List[Choice] = Depends(adapters.retrieve_choices),
) -> FastChoices:
    return FastChoices.from_qs(choices)


@router.get("/{c_id}")
def get_choice(choice: Choice = Depends(adapters.retrieve_choice)) -> FastChoice:
    return FastChoice.from_model(choice)
```

### `asgi.py`

Let's also add a FastAPI app into `mysite/asgi.py`.

```python
from fastapi import FastAPI
from polls.routers import choices_router
from polls.routers import questions_router

fastapp = FastAPI()
fastapp.include_router(questions_router, tags=["questions"], prefix="/question")
fastapp.include_router(choices_router, tags=["choices"], prefix="/choice")
```

### Run servers

First is to generate static files for `uvicorn` (you may still need [whitenoise](https://whitenoise.evans.io/en/stable/)):

```bash
python manage.py collectstatic --noinput
```

Now you can start FastAPI server by `uvicorn mysite.asgi:fastapp --reload` and start Django server by `uvicorn mysite.asgi:application --port 8001 --reload`.

Then you'll see your favorite FastAPI's OpenAPI documentation at `http://127.0.0.1:8000/docs/` and don't forget to check Django admin at `http://127.0.0.1:8001/admin/`.

## Conclusion

It's easier than we thought to integrate FastAPI and Django ORM, if you tactically separate the adapters of connecting Django ORM and Pydantic models, you'll get a clear and concise directory structure - easy to write and easy to maintain.

The whole project is also at my [Github](https://github.com/kigawas/fastapi-django), feel free to play with it.
