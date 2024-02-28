---
layout: post
comments: true
date: 2024-02-28
title: "[Spark 0] Parquet 파일은 어떤 과정을 거쳐 Spark DataFrame이 될까?"
subject: blog
category: Apache-Spark
tags:
  - Apache-Spark
---
# Parquet 파일이 어떤 과정을 거쳐 Spark DataFrame이 될까?

Spark는 다양한 파일 포맷을 지원합니다. 그 중 가장 많이 쓰이는 파일 포맷이 이 Parquet인데요.

저는 지금까지 Spark를 사용하며 단순히 `spark.read.format('parquet')` 처럼 Parquet 파일들을 읽어 DataFrame으로 만들고 사용하였습니다. 

하지만 여기서 궁금증이 발생한 것이죠. 어딘가에 저장된(여기서는 AWS S3로 가정합니다) Parquet 파일이 어떤 과정을 거쳐 Spark DataFrame 객체로 변환되는 것일까요?

위 과정들을 여러 방법을 통해 알아보고 발견한 내용들을 정리하여 저와 같은 궁금증을 가진 분들께 공유드리고자 이 글을 작성하게 되었습니다 :)

아래의 사전 지식이 이 글을 더 수월하게 읽도록 도와줍니다.
1. [[FileFormat - Parquet]]에 대한 이해 (열 지향 데이터, Serializer와의 관계)
2. [[Spark와 JVM의 관계]](DataFrame 또는 RDD가 JVM에서 어떻게 표현되는지)
	1. Serialize, Deserialize (Spark 객체로 변환되기까지의 단계에서 필수적으로 사용됨으로)
3. [[서버 간 네트워크를 통해 데이터가 이동하는 원리]] 이해
	1. ByteStream (다른 서버의 데이터를 Spark 서버로 가져올 때 사용됨, 추상화해도 될 것)
4. Spark의 Driver, Executor에 대한 이해

우선 우리가 흔히 사용하는 Parquet 파일을 DataFrame으로 읽어들이는 코드부터 봅시다.
```Python
spark.read.format('parquet').load('s3://some_path/some.parquet')
```

위 코드는 특정 path(여기에서는 AWS S3)에 저장된 Parquet 파일을 DataFrame 객체로 읽어들입니다.

아래와 같은 순서를 통해 DataFrame 객체가 생성됩니다.
1. 파일 위치 확인 및 접근:
	- Spark는 주어진 경로를 확인하고, 해당 경로에 저장된 Parquet 파일에 접근합니다.
2. Task 생성 및 분배:
	- Spark Driver는 주어진 경로의 파일을 읽기 위한 Task를 생성합니다.
	- 실질적으로 Parquet 파일을 읽기(Spark 서버로 load) 위해 Spark Executor에 수행할 Task를 분배합니다.
3. Parquet 파일 읽기:
	- 각 Executor는 주어진 경로의 Parquet 파일을 읽는 Task를 수행합니다.
	- 이 과정에서 각 Parquet 파일은 네트워크를 통해 ByteStream 형태로 각 Executor에 전달됩니다.
4. Parquet 파일 Deserialize:
	- 각 Executor는 Parquet Reader 객체를 사용하여 전달받은 ByteStream을 열 데이터로 Deserialze하는 Task를 수행합니다.
5. 스키마 적용:
	- Parquet 파일에 내장된 Schema 정보를 읽습니다.
	- 이를 바탕으로 Spark의 DataFrame에 적용될 Schema를 생성합니다.
6. DataFrame 생성:
	- Deserialize된 데이터는 Spark의 DataFrame 형태로 변환됩니다.
	- 이 DataFrame은 JVM 내에서 관리되는 Java 객체입니다.
7. 데이터 분산 처리:
	- 우리가 아는 것과 같이 DataFrame을 조작합니다.

각 단계들에 해당하는 Spark 코드를 첨부하면 보다 엔지니어링적으로 이해하기 쉬울 것 같습니다.

조만간 각 동작들이 어떠한 Spark 코드를 사용하여 수행되는지 함께 포스팅하겠습니다.

미흡한 점이 많은 간단한 글을 읽어주셔서 감사합니다 :)