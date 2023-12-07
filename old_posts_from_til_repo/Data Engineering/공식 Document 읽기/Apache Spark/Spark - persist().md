# PySpark.sql.DataFrame - persist

---

> DataFrame.persist(storageLevel) -> DataFrame
>
> Sets the storage level to persist the contents of the DataFrame acorss operations after the first time it is computed. This can only be used to assign new storage level if the DataFrame does not have a storage level set yet. If no storage level is specified defaults to `(MEMORY_AND_DISK_DESER)`
>
> **Parameters**: storageLevel: Storage level to set for persistence. Default is MEMORY_AND_DISK_DESER.
> **Returns**: DataFrame: Persisted DataFrame.	

이 함수는 Lazy evaluation을 따르기 때문에 `persist()` 후 바로 실행되지 않는다(action함수가 실행되어야 함). 지정한 Storage Level에 해당하는 저장소에 DataFrame을 저장한다(이 때, 이미 어떠한 Storage Level에 존재하는 경우 실행되지 않음). 메서드에 파라미터를 지정하지 않으면 기본값으로 `MEMORY_AND_DISK_DESER`가 사용된다.

---

### Example(1)

### Example

```python
>>> df = spark.range(1)
>>> df.persist()
# DataFrame[id: bigint]

df.explain()
# == Physical Plan ==
# InMemoryTableScan ...
```

- 특별히 어떠한 변수에 할당하지 않아도 된다. 
- Physical Plan을 살펴보면 `persist()`는 `InMemoryTableScan`로 표시된다. 

### Example(2)

```python
from pyspark.storagelevel import StorageLevel
df.persist(StorageLevel.DISK_ONLY)
# DataFrame[id: bigint]
```

- 보다 메모리 관리를 잘하고 싶다면 StorageLevel 객체에 대한 이해가 필요하다.

---

Reference: https://spark.apache.org/docs/latest/api/python/reference/pyspark.sql/api/pyspark.sql.DataFrame.persist.html#pyspark.sql.DataFrame.persist