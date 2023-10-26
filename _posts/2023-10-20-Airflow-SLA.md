# TODO: SLA를 Local 환경에서 테스트해보고, 이를 간단하게 글로 설명하자.
---
layout: post
comments: true
date: 2023-09-30
title: "Apache-Airflow - DAG - SLA"
description: "Apache Airflow의 DAG Level에서 SLA를 설정하는 방법에 대해 소개합니다."
subject: blog
category: "Apache-Airflow"
tags: [ "Apache-Airflow" ]
---
# Apache Airflow - DAG - SLA(System Level Agreement)
 
Airflow 공식 문서에서는 SLA를 아래와 같이 설명하고 있습니다.
> An SLA, or a Service Level Agreement, is an expectation for the maximum time a Task should be completed relative to the Dag Run start time. If a task takes longer than this to run, it is then visible in the “SLA Misses” part of the user interface, as well as going out in an email of all tasks that missed their SLA.
 
간단하게 설명하면, SLA는 하나의 Task가 실행되는 최대 시간을 지정하는 것입니다. 만약 Task가 지정된 시간보다 더 오래 걸린다면, 해당 Task는 SLA Misses로 표시되며, 이를 통해 사용자는 해당 Task가 예상보다 오래 걸렸음을 빠르게 파악할 수 있습니다.
 
## Code Example(Apache-Airflow 2.7.2)
---
```python
...
```