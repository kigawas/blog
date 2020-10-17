+++
title = "FastAPIとDjango ORMを組み合わせる"
date = "2020-08-21"
cover = ""
tags = ["Python", "web", "FastAPI", "Django"]
description = "FastAPIとDjango ORMを組み合わせるためのステップバイステップガイド"
showFullContent = false
+++

## 動機

最近 FastAPI が[破竹の勢い](https://star-history.t9t.io/#tiangolo/fastapi)で伸びているらしい。

Django から FastAPI に浮気をしたいけど、やはり Django やそのユーザーシステムを引き続き利用したい。欲張りに見えるが、実はそういういい都合もある。今回はどうやって Django ORM と FastAPI を組み合わせるかについて説明する。

> 当然デメリットもある。パーフォーマンス上は、asyncio 対応の ORM が使えなくなる。
>
> パーフォーマンス上げたいなら、別途 [orm](https://github.com/encode/orm)、[gino](https://github.com/python-gino/gino) などのモデルを作成するがよい。
>
> 因みに、Django ORM も asyncio に対応する[予定があるらしい](https://docs.djangoproject.com/en/3.1/topics/async/)。

## フォルダ構造

まず、フォルダ構成について話す。フォルダの内容は Django ドキュメントのチュートリアルに基づいて作成していい。詳細は[ここ](https://docs.djangoproject.com/en/3.1/intro/tutorial01/)。

```bash
django-admin startproject mysite
django-admin startapp poll
```

ファイルを生成した後、`views.py`と`models.py`を削除して、下記のように FastAPI 用のフォルダを用意する。

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

各フォルダの使い分け：

- `models`フォルダ：Django ORM
- `routers`フォルダ：FastAPI routers
- `schemas`フォルダ：FastAPI の Pydantic バリデータ
- `adapters`フォルダ：Django ORM を取得

ORM を使う FastAPI のウェブサービスにとって、ORM と Pydantic モデルの両方が存在する。いかに ORM を Pydantic モデルに変換するかというと、Pydantic の[ORM モード](https://pydantic-docs.helpmanual.io/usage/models/#orm-mode-aka-arbitrary-class-instances)を使うのは普通。

## データを用意

[Django ドキュメント](https://docs.djangoproject.com/en/3.1/intro/tutorial02/)を参考にして、データを入れてみる。

```python
>>> from polls.models import Choice, Question
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())
>>> q.save()
```

## FastAPI 導入

簡略化するため、下の例はフォルダの`__init__.py`に入れる。一部の`import`も省略する。

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
    return FastQuestion.from_orm(question)

@router.get("/")
def get_choices(
    choices: List[Choice] = Depends(adapters.retrieve_choices),
) -> FastChoices:
    return FastChoices.from_qs(choices)


@router.get("/{c_id}")
def get_choice(choice: Choice = Depends(adapters.retrieve_choice)) -> FastChoice:
    return FastChoice.from_orm(choice)
```

### `asgi.py`

`mysite/asgi.py`に、FastAPI の App の起動も追加。

```python
from fastapi import FastAPI
from polls.routers import choices_router
from polls.routers import questions_router

fastapp = FastAPI()
fastapp.include_router(questions_router, tags=["questions"], prefix="/question")
fastapp.include_router(choices_router, tags=["choices"], prefix="/choice")
```

### 立ち上げ

まず、`uvicorn`用のスタティックファイルを作成（[whitenoise](http://whitenoise.evans.io/en/stable/index.html)も必要）：

```bash
python manage.py collectstatic --noinput
```

FastAPI は`uvicorn mysite.asgi:fastapp --reload`で、Django は`uvicorn mysite.asgi:application --port 8001 --reload`で起動。

FastAPI の Doc は`http://127.0.0.1:8000/docs/`に、Django の admin 画面は`http://127.0.0.1:8001/admin/`にアクセス。

## まとめ

FastAPI と Django ORM の組み合わせは意外と簡単で、上手にインテグレーションの部分を分けると、すっきりとしたフォルダ構造もできるのである。

上のコードは筆者の[Github](https://github.com/kigawas/fastapi-django)にもある。
