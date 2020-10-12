# The integration of FastAPI and Django ORM

## Motivation

Recently FastAPI is [growing incredibly](https://star-history.t9t.io/#tiangolo/fastapi).

You may really want to move to FastAPI from Django, however, it's not that easy to give up Django and it's user system. 欲張りに見えるが、実はそういういい都合もある。今回はどうやって Django ORM と FastAPI を組み合わせるかについて説明する。

> 当然デメリットもある。パーフォーマンス上は、asyncio 対応の ORM が使えなくなる。
> パーフォーマンス上げたいなら、別途 [orm](https://github.com/encode/orm) などのモデルを作成するがよい。
> 因みに、Django ORM も asyncio に対応する[予定があるらしい](https://docs.djangoproject.com/en/3.1/topics/async/)。

## フォルダ構造

まず、フォルダ構成について話す。フォルダの内容は Django ドキュメントのチュートリアルに基づいて作成していい。詳細は[ここ](https://docs.djangoproject.com/en/3.1/intro/tutorial01/)。

```bash
$ django-admin startproject mysite
$ django-admin startapp poll
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
- `adapters`フォルダ：Django ORM を Pydantic バリデータに変換する

## データを用意

[Django ドキュメント](https://docs.djangoproject.com/en/3.1/intro/tutorial02/)を参考にして、データを入れてみる。

```python
>>> from polls.models import Choice, Question
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())
>>> q.save()
```

## FastAPI 導入

簡略化するため、下の例はフォルダの`__init__.py`に入れる。`import`も省略する。

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

`mysite/asgi.py`に、FastAPI の App の起動も追加。

```python
from fastapi import FastAPI
from polls.routers import router

fastapp = FastAPI()
fastapp.include_router(router, tags=["questions"], prefix="/question")
```

### 立ち上げ

まず、`uvicorn`用のスタティックファイルを作成：

```bash
$ python manage.py collectstatic --noinput
```

FastAPI は`uvicorn mysite.asgi:fastapp --reload`で、Django は`uvicorn mysite.asgi:application --port 8001 --reload`で起動。

FastAPI の Doc は`http://127.0.0.1:8000/docs/`に、Django の admin 画面は`http://127.0.0.1:8001/admin/`にアクセス。

## まとめ

FastAPI と Django ORM の組み合わせは意外と簡単で、上手にインテグレーションの部分を分けると、すっきりとしたフォルダ構造もできるのである。

上のコードは筆者の[Github](https://github.com/kigawas/fastapi-django)にもある。
