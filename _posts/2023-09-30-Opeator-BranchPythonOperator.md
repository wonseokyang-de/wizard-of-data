---
layout: post
title: Apache Airflow - Operator - BranchPythonOpeator
date: 2023-09-30
categories: Apache Airflow
---
안녕하세요. 이번 글은 Airflow에서 특정 조건을 통해 분기하는 Task를 만들기 위해 사용되는 `BranchPythonOperator` 사용법에 대해 정리해보려 합니다.

```python
class airflow.operators.python.BranchPythonOperator
```
<br>
Airflow 공식 문서에서 정의하는 BranchPythonOperator는 다음과 같습니다.
> It derives the PythonOperator and expects a Python function that returns a single task_id or list of task_ids to follow. The task_id(s) returned should point to a task directly downstream from {self}. All other “branches” or directly downstream tasks are marked with a state of skipped so that these paths can’t move forward. The skipped states are propagated downstream to allow for the DAG state to fill up and the DAG run’s state to be inferred.

간단하게 설명하자면, 기본적으로 위 Operator는 Python 함수를 python_callable 키워드 인자로 사용하며, Python 함수가 반환하는 task_id(s)에 해당하는 Downstream Task(s)를 실행하도록 지시하는 지시자역할을 수행합니다. task_id(s)로 표현한 것과 같이, 하나의 Task에 해당하는 task_id를 반환하면 하나만 실행되며, List로 묶인 여러 개에 해당하는 task_id를 반환하면 해당하는 여러 개의 Task들이 실행됩니다. 해당 함수에서 반환되지 않은 Downstream Task(s)는 state가 `skipped`로 처리됩니다.

### 예제 코드
---
```python
from airflow import DAG
from airflow.operators.dummy import DummyOperator
from airflow.operators.python import PythonBranchOperator


def branching_function(condition):
    if condition == 1:
        return 'branch_a'
    else:
        return 'branch_b'


with DAG(
    ...
) as dag:

    branching_task = PythonBranchOperator(
        task_id='branching',
        python_callable=branching_function,
        op_kwargs={
            'condition': 1
        }
    )
    
    a_task = DummyOperator(
        task_id='branch_a'
    )

    b_task = DummyOperator(
        task_id='branch_b'
    )

    (
        branching_task
        >> [  # BranchPythonOperator의 대상이 될 Downstream Taske들은 List로 정의합니다.
            a_task,
            b_task
        ]
    )
```
