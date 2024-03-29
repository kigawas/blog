+++
title = "The integration of FastAPI and Django ORM"
date = "2020-10-12"
cover = ""
tags = ["Python", "web", "FastAPI", "Django"]
description = "A step-by-step guide to integrating FastAPI and Django ORM"
showFullContent = false
+++

This is the English translation of the Japanese original post at [qiita](https://qiita.com/kigawas/items/80e48ccce98a35f65fff), with some modifications.

## Motivation

Recently FastAPI is [growing incredibly](https://star-history.t9t.io/#tiangolo/fastapi). It's blazingly fast and painless to develop, with [5~10x performance enhancement](https://www.techempower.com/benchmarks/#section=data-r20&hw=ph&test=fortune&l=zijzen-sf) over Django or Flask.

I really want to switch to FastAPI from Django, however, it's not that easy to give up Django and its self-sufficient user system as well as the admin page totally. I know it sounds greedy, but in fact there **is** such convenience. This time I'll show you how to integrate FastAPI and Django ORM simply and quickly.

> There's also a demerit undoubtedly. Django ORM is [not fully asynchronous](https://docs.djangoproject.com/en/dev/topics/async/#queries-the-orm) and this will hurt performance sometimes.
>
> If you'd like to improve, you may consider using [orm](https://github.com/encode/orm), [gino](https://github.com/python-gino/gino) or [sqlalchemy 1.4+](https://docs.sqlalchemy.org/en/14/orm/extensions/asyncio.html) to rewrite some logic.
>
> From Django 4.1, you can make [asynchronous queries](https://docs.djangoproject.com/en/dev/topics/db/queries/#async-queries) with Django ORM. Note that async transactions are not supported yet, and the PostgreSQL driver is [still `psycopg2`](https://docs.djangoproject.com/en/dev/ref/databases/#postgresql-notes).

## Directory structure

Let's talk about the directory structure first. You can just follow Django's [tutorial](https://docs.djangoproject.com/en/dev/intro/tutorial01/) to create the scaffold.

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
- `adapters`: The adapters to retrieve Django ORMs
- `schemas`: FastAPI Pydantic models

For a typical FastAPI application, there should be an ORM part and a Pydantic model/validator part. For how to convert ORMs to Pydantic models, normally we'd like to utilize the [ORM mode](https://pydantic-docs.helpmanual.io/usage/models/#orm-mode-aka-arbitrary-class-instances).

## Set up some data

Let's refer to the [Django documentation](https://docs.djangoproject.com/en/dev/intro/tutorial02/) and insert some data:

```python
>>> from polls.models import Choice, Question
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())
>>> q.save()
```

## FastAPI integration

For simplicity, I'll ignore some `import` statements.

### `schemas`

```python
from django.db import models
from pydantic import BaseModel as _BaseModel

class BaseModel(_BaseModel):
    @classmethod
    def from_orms(cls, instances: List[models.Model]):
        return [cls.from_orm(inst) for inst in instances]


class FastQuestion(BaseModel):
    question_text: str
    pub_date: datetime

    class Config:
        orm_mode = True


class FastQuestions(BaseModel):
    items: List[FastQuestion]

    @classmethod
    def from_qs(cls, qs):
        return cls(items=FastQuestion.from_orms(qs))


class FastChoice(BaseModel):
    question: FastQuestion
    choice_text: str

    class Config:
        orm_mode = True


class FastChoices(BaseModel):
    items: List[FastChoice]

    @classmethod
    def from_qs(cls, qs):
        return cls(items=FastChoice.from_orms(qs))
```

### `adapters`

We can make async queries from Django 4.1.

```python
ModelT = TypeVar("ModelT", bound=models.Model)


async def retrieve_object(model_class: Type[ModelT], id: int) -> ModelT:
    instance = await model_class.objects.filter(pk=id).afirst()
    if not instance:
        raise HTTPException(status_code=404, detail="Object not found.")
    return instance


async def retrieve_question(q_id: int = Path(..., description="get question from db")):
    return await retrieve_object(Question, q_id)


async def retrieve_choice(c_id: int = Path(..., description="get choice from db")):
    return await retrieve_object(Choice, c_id)


async def retrieve_questions():
    return [q async for q in Question.objects.all()]


async def retrieve_choices():
    return [c async for c in Choice.objects.all()]
```

### `routers`

#### `routers/__init__.py`
```python
from .choices import router as choices_router
from .questions import router as questions_router

__all__ = ("register_routers",)


def register_routers(app: FastAPI):
    app.include_router(questions_router)
    app.include_router(choices_router)
```
#### `routers/choices.py`

```python
router = APIRouter(prefix="/choice", tags=["choices"])


@router.get("/", response_model=FastChoices)
def get_choices(
    choices: List[Choice] = Depends(adapters.retrieve_choices),
) -> FastChoices:
    return FastChoices.from_qs(choices)


@router.get("/{c_id}", response_model=FastChoice)
def get_choice(choice: Choice = Depends(adapters.retrieve_choice)) -> FastChoice:
    return FastChoice.from_orm(choice)
```

#### `routers/questions.py`

```python
router = APIRouter(prefix="/question", tags=["questions"])


@router.get("/", response_model=FastQuestions)
def get_questions(
    questions: List[Question] = Depends(adapters.retrieve_questions),
) -> FastQuestions:
    return FastQuestions.from_qs(questions)


@router.get("/{q_id}", response_model=FastQuestion)
def get_question(
    question: Question = Depends(adapters.retrieve_question),
) -> FastQuestion:
    return FastQuestion.from_orm(question)
```

### `asgi.py`

Let's also add a FastAPI app into `mysite/asgi.py`.

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles

fastapp = FastAPI()


def init(app: FastAPI):
    from polls.routers import register_routers

    register_routers(app)

    if settings.MOUNT_DJANGO_APP:
        app.mount("/django", application)  # type:ignore
        app.mount("/static", StaticFiles(directory="staticfiles"), name="static")


init(fastapp)
```

### Run servers

First is to generate static files for `uvicorn` (you may still need [`whitenoise`](https://whitenoise.evans.io/en/stable/) if you don't mount the Django app with FastAPI):

```bash
python manage.py collectstatic --noinput
```

Now you can start FastAPI server by `uvicorn mysite.asgi:fastapp --reload` and start Django server by `uvicorn mysite.asgi:application --port 8001 --reload`.

Then you'll see your favorite FastAPI's OpenAPI documentation at `http://127.0.0.1:8000/docs/` and don't forget to check Django admin at `http://127.0.0.1:8001/admin/`.

If you just need one ASGI app, the Django app can be mounted on FastAPI app:

```python
# in mysite/settings.py

MOUNT_DJANGO_APP = True
```

Then the Django admin page can be found at `http://localhost:8000/django/admin`.

## Conclusion

It's easier than we thought to integrate FastAPI and Django ORM, if you tactically separate the adapters of connecting Django ORM and Pydantic models, you'll get a clear and concise directory structure - easy to write and easy to maintain.

The whole project is also at my [Github](https://github.com/kigawas/fastapi-django), feel free to play with it.
