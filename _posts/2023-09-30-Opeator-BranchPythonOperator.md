---
layout: post
title: Apache-Airflow - Operator - BranchPythonOperator
date: 2023-09-30
categories: Apache-Airflow
---
안녕하세요. 
이번 글은 Airflow에서 특정 조건에 맞는 Task를 실행하기 위해 Python 함수와 함께 사용되는 `BranchPythonOperator` 사용법에 대해 설명합니다.
 
Airflow 공식 문서에서 정의하는 `BranchPythonOperator`는 다음과 같습니다.
 
```python
class airflow.operators.python.BranchPythonOperator
```
 
> It derives the PythonOperator and expects a Python function that returns a single task_id or list of task_ids to follow. The task_id(s) returned should point to a task directly downstream from {self}. All other “branches” or directly downstream tasks are marked with a state of skipped so that these paths can’t move forward. The skipped states are propagated downstream to allow for the DAG state to fill up and the DAG run’s state to be inferred.
 
간단하게 위 Operator를 간단하게 설명하면, Python 함수를 python_callable 키워드 인자로 사용하여 해당 함수가 반환하는 문자열과 같은 task_id(s)에 해당하는 Downstream Task(s)가 실행되도록 합니다. 
task_id(s)와 Downstream Task(s)로 표현한 이유는 해당 Operator를 사용하여 생성한 Task가 List로 묶인 task_id들을 반환하면 Downstream으로 해당하는 여러 개의 Task들이 실행되도록 합니다. 
해당 함수에서 반환되지 않은 Downstream Task(s)는 state가 `skipped`로 처리됩니다.
<br>

### Code Example(Apache-Airflow 2.2.2)
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
			'condition': 1,
		}
	)

	a_task = DummyOperator(
		task_id='branch_a'
	)

	b_task = DummyOperator(
		task_id='branch_b'
	)

	branching_task >> [a_task, b_task]
```
