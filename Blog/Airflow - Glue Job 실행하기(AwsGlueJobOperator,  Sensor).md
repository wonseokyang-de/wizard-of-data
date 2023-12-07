# Airflow - Glue Job 실행하기(AwsGlueJobOperator, Sensor)

- 목차를 쓰면서, 내가 생각한 독자에게 타게팅이 되는 글인지 생각해보기
- 소제목 넣기
- 들어가는 글 다듬기

### INDEX

1. 이 글을 쓰게 된 이유
2. 현재 사용하고 있는 Glue Job 실행 관련 코드
3. 

현재 우리 팀에서 Data Pipeline을 생성할 때 사용하는 AWS 서비스는 MWAA(2.2.2)이다.

MWAA(이하 Airflow)에서는 AWS의 Glue Job을 실행하기 위한 Operator와 Sensor 등을 기본으로 포함하고 있는데, 그 중 providers 라이브러리에 포함되어 있는 AwsGlueJobOperator, AwsGlueJobSensor를 사용한다.

(AwsGlue~ 라는 이름이 붙은 Operator 및 Sensor는 `apache-airflow-providers-amazon` 패키지의 `2.6.0` 버전부터 **deprecated**되었고 Glue~ 라는 이름이 붙은 것들을 사용하도록 권장하고 있다. 우리 팀에서 사용하는 Airflow는 위에서 설명했듯이 `2.2.2` 버전을 사용하며, AWS에서 설정한 `apache-airflow-providers-amazon` 의 버전은 `2.4.1`이므로 Glue~ 라는 이름이 붙은 Operator 등을 사용할 수 없어 AwsGlue~ 를 사용하고 있다. — 버전 올리고 싶다)

![Apache Airflow Docs: apache-airflow-providers-amazon(2.6.0)](Airflow%20-%20Glue%20Job%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5(AwsGlueJobOperator,%20%20bfa9c13f0f2844c49f3e4a0b3ef31012/Screenshot_2023-05-01_at_1.36.59_PM.png)

Apache Airflow Docs: apache-airflow-providers-amazon(2.6.0)

현재 Glue Job을 실행하기 위해 사용하고 있는 코드는 다음과 같다.

```python
def create_glue_job_task_group(*, job_name, script_args, **kwargs):

    with TaskGroup(group_id=job_name) as task_group:
        run_glue_job = AwsGlueJobOperator(
            task_id='run_glue_job',
            job_name=job_name,
            script_args=script_args,
            **kwargs
        )
        get_glue_job_status = AwsGlueJobSensor(
            task_id='get_glue_job_status',
            job_name=job_name,
            run_id=run_glue_job.output,
        )
        
        run_glue_job >> get_glue_job_status

    return task_group
```

위 코드를 이해할 수 있으려면 아래의 요소에 대해서 알고 있어야 한다.

- Airflow - TaskGroup
- Python - **kwargs

위 코드에 대해 간략히 설명하자면, `create_glue_job_task_group()`은 `AwsGlueJobOperator`와 `AwsGlueJobSensor`가 하나의 `TaskGroup`으로 묶고 해당 `TaskGroup`을 반환하는 함수이다.

`AwsGlueJobOperator`는 `create_glue_job_task_group()`를 호출할 때 전달받은 파라미터 `job_name`에 해당하는 Glue Job을 실행시키며, 이 Job을 실행시킬 때 동일 함수의 파라미터 `script_args`를 파라미터로 전달한다.

위 코드는 아래와 같이 호출하여 사용한다.

```python
brandi_raw_impression_processor = create_glue_job_task_group(
    job_name='glue_job_name',
    script_args={
        '--target_date': '{{ data_interval_end }}',
        '--s3_input_bucket': 's3_input_bucket',
        '--s3_output_bucket': 's3_output_buket'
    },
    retry_limit=1 # **kwargs를 통해 전달되는 param(BaseOperator)
)
```

하지만, 회사에서 정신적 사수로 생각하고 있는 어느 한 분께서 이 코드를 보시더니 `AwsGlueJobSensor`의 파라미터 중 `run_id`에 전달되는 `run_glue_job.output` 이 어떻게 동작되는 것이냐는 의문을 내게 주셨다. 

나는 이 코드를 생성할 때 잘 돌아가니 가만히 두었는데, 이 부분이 프로답지 못하고 안일했다는 생각이 들었다.

그러므로, 이 코드가 내부적으로 어떻게 작성되어 있길래 `run_glue_job.output` 으로 앞선 `AwsGlueJobOperator` 의 반환값을 가져올 수 있는지 확인하고자 이 글을 쓰게 되었다.

- 내부 구조를 이해하기 위한 순서
    1. `AwsGlueJobOperator`의 반환값이 어떤 것인지
    2. `AwsGlueJobOperator`의 상위 객체인 `BaseOperator`의 `xcom_push()` 과정은 어떻게 이루어 지는지
    3. `AwsGlueJobOperator`의 반환값을 `var_name.output`으로 어떻게 가져올 수 있는지

### 1. `AwsGlueJobOperator` 의 반환값이 어떤 것인지

`AwsGlueJobOperator.excute()` 의 반환 값은 `glue_job_run['JobRunId']` 이다. 

`AwsGlueJobOperator`에서 Glue Job을 실행시키는 로직은 내부적으로 `AwsGlueHook.initialize_job()`을 통해 실행되며 Operator를 호출할 때 전달받은 `wait_for_completion` 으로 실행하고자 하는 Glue Job의 실행 결과까지 기다릴지에 대한 여부를 결정한다. (`wait_for_completion`의 기본 값은 `True`)

wait_for_completion 관련 로직은 아래와 같다.(providers-amazon==2.4.1)

AwsGlueJobOperator.wait_for_completion → AwsGlueJobHook.job_completion() → AwsGlueJobHook.get_job_state()

아래는 AwsGlueJobOperator에서 Glue Job을 실행하는 부분에 대한 실제 로직들이다(log부분은 제거함).

- AwsGlueJobOperator의 wait_for_completion을 사용하는 부분

```python
if self.wait_for_completion:
		glue_job_run = glue_job.***job_completion***(self.job_name, glue_job_run['JobRunId'])
```

- AwsGlueJobHook의 job_completion()의 내부 로직
    - job_completion
      
        ```python
        def job_completion(self, job_name: str, run_id: str) -> Dict[str, str]:
            failed_states = ['FAILED', 'TIMEOUT']
            finished_states = ['SUCCEEDED', 'STOPPED']
        
            while True:
                job_run_state = self.***get_job_state***(job_name, run_id)
                if job_run_state in finished_states:
                    return {'JobRunState': job_run_state, 'JobRunId': run_id}
                if job_run_state in failed_states:
                    job_error_message = "Exiting Job " + run_id + " Run State: " + job_run_state
                    raise AirflowException(job_error_message)
                else:
                    time.sleep(self.JOB_POLL_INTERVAL)
        ```
        
    - get_job_state
      
        ```python
        def get_job_state(self, job_name: str, run_id: str) -> str:
            glue_client = self.get_conn()
            job_run = glue_client.***get_job_run***(JobName=job_name, RunId=run_id, PredecessorsIncluded=True)
            job_run_state = job_run['JobRun']['JobRunState']
            return job_run_state
        ```
        
    
    어찌됐든, AwsGlueJobOperator.execute()의 최종 반환값은 `return glue_job_run['JobRunId']` 이다.
    
    ### 2. `AwsGlueJobOperator`의 상위 객체인 `BaseOperator`의 `xcom_push()` 과정은 어떻게 이루어 지는지
    
    이 부분이 이번 글을 쓰게 된 가장 큰 핵심 요소이다. 
    
    AwsGlueJobOperator는 내부적으로 AwsGlueJobHook을 사용하여 Glue Job을 실행시킨다. 그렇기에 AwsGlueJobHook라는 Class의 내부 구조에 대한 이해가 있어야 최종적으로 찾으려고 한 xcom_push()관련 메서드를 찾는 데 한결 수월해질 것이다.
    
    `AwsGlueJobHook`을 알아보기 전에 먼저 `AwsGlueJobOperator`에서 `AwsGlueJobHook`이 어떻게 사용되는지 확인할 필요가 있다.
    
    아래의 예시 코드와 같이, `AwsGlueJobOperator`는  `AwsGlueJobHook`을 사용하여 `glue_job` 이란 이름을 가진 객체를 만들고, 차례로 `initialize_job()` → `job_completion()`을 거쳐 실행된 Glue Job의 `run_id --glue_job_run['JobRunId']`를 반환한다.
    
    ```python
    glue_job = AwsGlueJobHook( # <- Hook을 이용하여 Glue Job을 실행하기 위한 인스턴스를 생성
        job_name=self.job_name,
        desc=self.job_desc,
        concurrent_run_limit=self.concurrent_run_limit,
        script_location=s3_script_location,
        retry_limit=self.retry_limit,
        num_of_dpus=self.num_of_dpus,
        aws_conn_id=self.aws_conn_id,
        region_name=self.region_name,
        s3_bucket=self.s3_bucket,
        iam_role_name=self.iam_role_name,
        create_job_kwargs=self.create_job_kwargs,
    )
    
    # ...
    
    glue_job_run = glue_job.initialize_job(self.script_args, self.run_job_kwargs) # 첫 번째로 알아볼 메서드
        if self.wait_for_completion:
            glue_job_run = glue_job.job_completion(self.job_name, glue_job_run['JobRunId']) # 두 번째로 알아볼 메서드
            self.log.info(
                "AWS Glue Job: %s status: %s. Run Id: %s",
                self.job_name,
                glue_job_run['JobRunState'],
                glue_job_run['JobRunId'],
            )
        else:
            self.log.info("AWS Glue Job: %s. Run Id: %s", self.job_name, glue_job_run['JobRunId'])
        return glue_job_run['JobRunId']
    ```
    
    먼저 `AwsGlueJobHook.initialize_job()`의 내부 로직을 분석하여 어떤 역할을 수행하는 메서드인지 확인해보자.
    
    ```python
    def initialize_job(
        self,
        script_arguments: Optional[dict] = None,
        run_kwargs: Optional[dict] = None,
    ) -> Dict[str, str]:
    
        glue_client = self.get_conn() # 1. boto3를 사용하여 Client 객체 생성
        script_arguments = script_arguments or {}
        run_kwargs = run_kwargs or {}
    
        try:
            job_name = self.get_or_create_glue_job() # 2. Hook 객체를 인스턴스화할 때 전달받은 job_name과 같은 Job이 존재하는지 확인하고, 존재하지 않다면 인스턴스화할 때 전달받은 args로 신규 Job을 생성
            job_run = glue_client.start_job_run(JobName=job_name, Arguments=script_arguments, **run_kwargs) # 3. boto3를 통해 생성한 glue_client를 통해 job_name에 해당하는 Job을 실행
            return job_run
        except Exception as general_error:
            self.log.error("Failed to run aws glue job, error: %s", general_error)
            raise
    ```
    
    `initialize_job()`은 `boto3`를 통해 Glue Job을 새로 만들거나 실행하기 위한 Client를 생성하고, 생성된 Client를 사용하여 기존 Job List에 앞서 전달한 `job_name`이란 이름의 Glue Job이 이미 존재하는지 확인하고 없다면 새로 만들어 실행하는 메서드이다.
    
    이후 `if`문을 거쳐 `self.wait_for_completion` 값이 `True`라면 실행되는 `glue_client.start_job_run()`에 대해 알아보자.
    
    ```python
    def job_completion(self, job_name: str, run_id: str) -> Dict[str, str]:
    
        failed_states = ['FAILED', 'TIMEOUT']
        finished_states = ['SUCCEEDED', 'STOPPED']
    
        while True:
            job_run_state = self.get_job_state(job_name, run_id) # 1. boto3.client를 통해 Job의 State를 전달받음
            if job_run_state in finished_states: # 2. Job의 State가 SUCCEEDED, STOPPED라면 실행됨
                self.log.info("Exiting Job %s Run State: %s", run_id, job_run_state)
                return {'JobRunState': job_run_state, 'JobRunId': run_id}
            if job_run_state in failed_states: # 3. Job의 State가 FAILED, TIMEOUT이라면 실행됨
                job_error_message = "Exiting Job " + run_id + " Run State: " + job_run_state
                self.log.info(job_error_message)
                raise AirflowException(job_error_message)
            else: # 4. failed_states, finished_states에 정의된 값 이외(e.g. RUNNING)의 값이라면 JOB_POLL_INTERVAL만큼 sleep()하며 while문을 계속 진행함
                self.log.info(
                    "Polling for AWS Glue Job %s current run state with status %s", job_name, job_run_state
                )
                time.sleep(self.JOB_POLL_INTERVAL)
    ```
    
    우선, AwsGlueJobOperator에는 `glue_job_run['JobRunId']` 을 반환하는 부분만 존재하고 xcom_push()과 관련된 내용은 없다. 그러므로 AwsGlueJobOperator의 상위 class인 BaseOperator에서 xcom_push() 관련 메서드를 찾아보고자 한다.
    
    ~~→ Airflow의 BaseOperator는 MetaClass를 사용한다. 하지만 나는 MetaClass에 대해 정확한 지식이 없으므로 우선 넘어가고자 한다.~~