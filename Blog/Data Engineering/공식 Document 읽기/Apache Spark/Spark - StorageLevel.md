# Spark - StorageLevel

---

> `class pyspark.StorageLevel(useDisk: bool, useMemory: bool, useOffHeap: bool, deserialized: bool, replication: int = 1)`
>
> Flags for controlling the storage of an RDD. Each StorageLevel records whether to use memory, whether to keep the data in memory in a JAVA-specific serialized format, and whether to replicate the RDD partitions on multiple nodes. Also contains static constants for some commonly used storage levels, MEMORY_ONLY. Since the data is always serializaed on the Python side, all the constants use the serialized formats.
>
> Attributes:
>
> - DISK_ONLY: 데이터를 디스크에 저장한다(복사본 1개).
> - MEMORY_AND_DISK: 데이터를 메모리에 먼저 저장하려고 시도한다. 만약 메모리가 부족하다면 초과하는만큼의 데이터를 디스크에 저장한다.
> - MEMORY_AND_DISK_DESER: 데이터를 역직렬화(DESER)된 형태로 저장한다.
> - MEMORY_ONLY: 데이터를 메모리에 저장한다. 만약 메모리가 부족하면 일부 데이터(파티션)는 저장되지 않고 버려지며, 버려진 데이터(파티션)은 나중에 필요할 때 다시 계산한다.
> - NONE: 어떠한 데이터도 저장하지 않도록 한다. 이 옵션은 특수한 경우에 사용되며, 해당 데이터가 매번 다시 계산되어야 하기 때문에 성능에 부정적인 영향을 미치기 쉽다.
> - OFF_HEAP: 데이터를 오프-힙 메모리에 저장하려고 시도한다.



- 위 Attributes를 이해하기 위한 Spark 내부 코드 중 일부

```scala
class StorageLevel private(
    private var _useDisk: Boolean,
    private var _useMemory: Boolean,
    private var _useOffHeap: Boolean,
    private var _deserialized: Boolean,
    private var _replication: Int = 1) { ... }

object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val DISK_ONLY_3 = new StorageLevel(true, false, false, false, 3)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(true, true, true, false, 1)
  ...
}
```



### words

---

1. 오프-힙(Off-Heap) 메모리:
    - JVM의 힙 메모리 외부에 할당되는 메모리
    - 힙 메모리의 여러 제약사항들로 인해 큰 데이터를 다루거나 메모리 관리를 더 세밀하게 제어하려는 경우 사용
    - GC의 관리 범위를 벗어나므로 Garbage Collection으로 인한 오버헤드가 없음
    - 오프-힙 메모리는 운영체제의 메모리 관리 체계에 의해 관리되며, 직접 메모리 관리를 해야하므로 사용이 비교적 복잡함
    - **Spark에서의 OFF_HEAP 옵션은 대량의 데이터를 빠르게 처리하면서 메모리 관리를 효율적으로 수행하기 위해 사용**
2. JVM 힙(Heap) 메모리:
    - Java 객체들이 저장되는 공간
    - JVM의 GC에 의해 관리
    - GC의 작업은 시스템 성능에 영향을 미칠 수 있으며, 힙 메모리의 크기는 물리적 메모리의 한계와 JVM 설정에 의해 제한된다.