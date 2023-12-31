> Airflow Docs의 내용들을 해석하면서 필요한 정리들을 작성합니다.
 
### Executor란 무엇인가?
Executor는 Task instance가 실행되는 로직을 결정하는 Airflow의 Core concept 중 하나입니다.
 
Airflow의 설정파일 중 [core] 섹션에서 `executor = {사용하려는 Executor}`로 사용할 Executor를 설정할 수 있습니다.
 
### Executor의 2가지 Type
---
Executor는 크게 2가지의 Type으로 나뉩니다.
1. Local
	1. Debug Executor(deprecated)
	2. Local Executor
	3. Sequential Executor
2. Remote
	1. Celery Executor
	2. CeleryKubernetes Executor
	3. Dask Executor
	4. Kubernetes Executor
	5. LocalKubernetes Executor
 
Local Executor로 분류되는 경우는 Executor가 Scheduler process에서 동작하며, Remote Executor로 분류되는 Executor의 경우 보통 [Worker의 Pool]에서 동작합니다.
 
? Worker의 Pool이 무엇이냐.
> Pool은 Airflow의 여러 Worker 간 Task를 분배하는 데 사용되는 자원 할당 메커니즘입니다.
> Airflow에서 "pool" 이라는 용어는 일반적으로 동시에 실행할 수 있는 Task의 수를 제한하는 데 사용됩니다. 이는 Airflow의 자원 관리와 작업 스케줄링에 중요한 역할을 합니다.
> - Pool의 기능
> 	1. 동시 실행 제한: Pool은 동시에 실행할 수 있는 Task의 최대 수를 정의하여 Airflow가 관리하는 리소스(e.g. DB Connection, Memory, CPU...)를 효율적으로 사용하도록 돕습니다.
> 	2. 자원 관리: 특정 Task가 많은 리소스를 사용할 경우, 이 특정 Task를 별도의 Pool에 할당하여 다른 Task의 실행에 영향을 주지 않도록 합니다.
> 	3. 우선 순위 및 분류: 서로 다른 Pool을 사용하여 Task에 우선순위를 부여하거나, 리소스 사용을 분류할 수 있습니다. 예를 들어, 긴급한 Task를 위한 Pool과 일반적인 Task를 위한 별도의 Pool을 만들 수 있습니다.
 
또한 아래와 같은 note가 있어 함께 공유합니다.
> New Airflow users may assume they need to run a separate executor process using one of the Local or Remote Executors. This is not correct. The executor logic runs _inside_ the scheduler process, and will run the tasks locally or not depending the executor selected.
> -> Executor의 로직은 Scheduler의 Process에서 실행된다! 그렇기에 따로 Executor Process를 실행할 필요가 없다!
 
### Writing Your Own Executor
---
Airflow의 각 Executor들은 전부 BaseExecutor라는 인터페이스를 사용합니다. 이 중 중요하다고 판단되어 Document에 명시되어 있는 Method들을 정리합니다. (*이 Method들은 왠만하면 overriding하지 말라고 합니다.*)
- **hearbeat**: Airflow Scheduler Job은 주기적으로 Executor에서 heartbeat를 호출합니다. 이 heartbeat는 Airflow Scheduler와 Executor 사이에서 동작하는 주요 포인트가 됩니다. 또한 heartbeat는 몇몇의 지표들을 업데이트하며, 새로 Queue에 추가된 Task를 실행하도록 Triggering하며, 실행중 혹은 완료된 작업의 상태를 업데이트합니다.
- **queue_command**: 
- **get_event_buffer**: Executor가 실행중인 Task instance의 현재 상태를 검색합니다.
- **has_task**: 이미 Queue에 있거나 실행중인 특정 Task instance가 있는지 확인합니다.
- **send_callback**: 설정한 callback들을 Executor로 보냅니다.
 
### Mandantory Methods to Implement
---
- **sync**
- **execute_async**
 
### Optional Interface Methods to Implement
---
- **start**
- end
- terminate
- cleanup_stuck_queued_tasks
- try_adopt_task_instances
- get_cli_commands
- get_task_log
