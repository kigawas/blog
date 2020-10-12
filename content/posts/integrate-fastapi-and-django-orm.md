+++
title = "The integration of FastAPI and Django ORM"
date = "2020-10-12"
cover = ""
tags = ["Python", "web", "fastapi", "django"]
description = "A step-by-step guide to integrate FastAPI and Django ORM"
showFullContent = false
+++

This is the English transation of the Japanese original post at [qiita](https://qiita.com/kigawas/items/80e48ccce98a35f65fff).

## Motivation

Recently FastAPI is [growing incredibly](https://star-history.t9t.io/#tiangolo/fastapi). It's blazingly fast and painless to usage, with [5~10x performace enhancement](https://www.techempower.com/benchmarks/#section=data-r19&hw=ph&test=fortune&l=zijzen-1r) over Django or Flask.

I really want to switch to FastAPI from Django, however, it's not that easy to give up Django and it's user system. I know it sounds greedy, but in fact there is such convenience. This time I'll show you how to integrate FastAPI and Django ORM simply and quickly.

> There's also a demerit undoubtedly. You cannot use asyncio ORM and this will hurt performance.
> If you'd like to improve it, you may think about using [orm](https://github.com/encode/orm) or [gino](https://github.com/python-gino/gino) to rewrite some logic.
> By the way, Django ORM is also [having a plan](https://docs.djangoproject.com/en/3.1/topics/async/) for supporting asyncio.

## Directory structure

Let's talk about the folder structure first. You can just follow Django's [tutorial](https://docs.djangoproject.com/en/3.1/intro/tutorial01/) to create the scaffold.

```bash
django-admin startproject mysite
django-admin startapp poll
```

After you're done, delete `views.py` and `models.py` and prepare the folders for FastAPI like below.

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
- `adapters`: The adapters to convert Django ORM to Pydantic validator

## Set up some data

Let's refer the [Django documentation](https://docs.djangoproject.com/en/3.1/intro/tutorial02/) and insert some data:

```python
>>> from polls.models import Choice, Question
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())
>>> q.save()
```

## FastAPI integration

For simplicity, the codes below are all in the `__init__.py` file, and I'll also ignore `import` statements.

### `schemas`

```python
class FastQuestion(BaseModel):
    question_text: str
    pub_date: datetime

    @classmethod
    def from_model(cls, instance: Question):
        return cls(
            id=instance.id,
            question_text=instance.question_text,
            pub_date=instance.pub_date,
        )


class FastQuestions(BaseModel):
    items: List[FastQuestion]

    @classmethod
    def from_qs(cls, qs):
        return cls(items=[FastQuestion.from_model(i) for i in qs])
```

### `adapters`

```python
ModelT = TypeVar("ModelT", bound=models.Model)


def retieve_object(model_class: Type[ModelT], id: int) -> ModelT:
    instance = model_class.objects.filter(pk=id).first()
    if not instance:
        raise HTTPException(status_code=404, detail="Object not found.")
    return instance


def retrieve_question(
    q_id: int = Path(..., description="retrive question from db")
) -> Question:
    return retieve_object(Question, q_id)


def retrieve_questions():
    return Question.objects.all()
```

### `routers`

```python
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

```

### `asgi.py`

Let's also add FastAPI app into `mysite/asgi.py`.

```python
from fastapi import FastAPI
from polls.routers import router

fastapp = FastAPI()
fastapp.include_router(router, tags=["questions"], prefix="/question")
```

### Run servers

The first is to generate static files for `uvicorn`:

```bash
python manage.py collectstatic --noinput
```

Start FastAPI by`uvicorn mysite.asgi:fastapp --reload` and start Django server by `uvicorn mysite.asgi:application --port 8001 --reload`.

Then you can see the FastAPI's OpenAPI documentation at `http://127.0.0.1:8000/docs/` and don't forget to check Django admin at `http://127.0.0.1:8001/admin/`.

## Conclusion

It's easier than we thought to integrate FastAPI and Django ORM, if you separate the adapters of connecting Django ORM and Pydantic models, you'll get a clear and concise directory structure - easy to write and easy to maintain.

The whole project above is also at my [Github](https://github.com/kigawas/fastapi-django), feel free to play with it.
