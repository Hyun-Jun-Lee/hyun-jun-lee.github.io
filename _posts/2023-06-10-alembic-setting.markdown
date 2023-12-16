---
title: "alembic setting" #Article title.
date: 2023-06-10
category: [PYTHON, PYTHON] #One, more categories or no at all.
tag: [python]
---

<aside>
💡 db를 mysql을 사용할 때, alembic `env.py` 에서 sqlalchemy 연결을 해주는 과정에서 `mysqlclient` 라이브러리가 필요(mysqlclinet 설치에러난다면 →)

</aside>

```python
pip install alembic

# root 디렉토리에 하는 것을 권장
alembic init alembic
```

## alembic.ini 설정

```python
...
...
[alembic]
# init으로 만든 폴더 위치
script_location = alembic

# env.py에서 설정해야 python 활용해서 환경변수 import 가능
sqlalchemy.url = ..
```

## alembic/env.py 설정

- 최상단에 sys.path 등록

```python
import os, sys

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.realpath(__file__))))
parent_dir = os.path.abspath(os.path.join(os.getcwd(), "..", ".."))
sys.path.append(parent_dir)
```

- autogernate 설정 및 sqlalchemy 연결

```python
from alembic import context
from db.base import Base
from db import models

def get_url():
    from app.config import (
        DATABASE_HOST,
        DATABASE_NAME,
        DATABASE_PASSWORD,
        DATABASE_PORT,
        DATABASE_USER,
    )

    return f"mysql://{DATABASE_USER}:{DATABASE_PASSWORD}@{DATABASE_HOST}:{DATABASE_PORT}/{DATABASE_NAME}?charset=utf8"

config = context.config
# Interpret the config file for Python logging.
# This line sets up loggers basically.
fileConfig(config.config_file_name)

# autogernate 적용 위한 세팅
target_metadata = Base.metadata
```

- migrations 함수 수정

```python
def run_migrations_offline() -> None:
		...
		...
		...

def run_migrations_online():
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
		# create_engine 대신에 기존 engin을 써도 되지 않을라나
    connectable = create_engine(
        get_url(),
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata, compare_type=True
        )

        with context.begin_transaction():
            context.run_migrations()
```

- Full Code

```python
import os, sys

sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.realpath(__file__))))
parent_dir = os.path.abspath(os.path.join(os.getcwd(), "..", ".."))
sys.path.append(parent_dir)
from logging.config import fileConfig
from sqlalchemy import create_engine
from sqlalchemy import pool
from alembic import context
from db.base import Base
from db import models

def get_url():
    from app.config import (
        DATABASE_HOST,
        DATABASE_NAME,
        DATABASE_PASSWORD,
        DATABASE_PORT,
        DATABASE_USER,
    )

    return f"mysql://{DATABASE_USER}:{DATABASE_PASSWORD}@{DATABASE_HOST}:{DATABASE_PORT}/{DATABASE_NAME}?charset=utf8"

# this is the Alembic Config object, which provides
# access to the values within the .ini file in use.
config = context.config
# Interpret the config file for Python logging.
# This line sets up loggers basically.
fileConfig(config.config_file_name)

# autogernate 적용 위한 세팅
target_metadata = Base.metadata

# other values from the config, defined by the needs of env.py,
# can be acquired:
# my_important_option = config.get_main_option("my_important_option")
# ... etc.

def run_migrations_offline() -> None:
    """Run migrations in 'offline' mode.

    This configures the context with just a URL
    and not an Engine, though an Engine is acceptable
    here as well.  By skipping the Engine creation
    we don't even need a DBAPI to be available.

    Calls to context.execute() here emit the given string to the
    script output.

    """
    url = get_url()
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    """Run migrations in 'online' mode.

    In this scenario we need to create an Engine
    and associate a connection with the context.

    """
    configuration = config.get_section(config.config_ini_section)
    configuration["sqlalchemy.url"] = get_url()
    connectable = create_engine(
        get_url(),
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection, target_metadata=target_metadata, compare_type=True
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```