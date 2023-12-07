# Python - Logger

로깅은 프로그램이 실행될 때 발생하는 이벤트를 추적하는 수단이다. 개발자는 코드에 logging을 추가하여 특정 이벤트가 발생했음을 나타낸다. python의 logging 라이브러리를 사용하면 이벤트에 중요도를 부여할 수 있다.

logging은 간단한 함수를 제공한다. 

- debug()
- info()
- warning()
- error()
- critical()

| 수행하려는 작업                                              | 최적의 도구                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| 명령행 스크립트 또는 프로그램의 일반적인 사용을 위한 콘솔 출력 표시 | `print()`                                                    |
| 프로그램의 정상 작동 중 발생하는 이벤트 출력(가령 상태 모니터링이나 결함 조사 등) | `logging.info()` 또는 진단 목적의 아주 자세한 출력의 경우 `logging.debug()` |
| 특정 실행시간 이벤트와 관련한 에러를 보고                    | `raise Exception`                                            |
| 예외를 발생시키지 않고 에러의 억제를 보고(가령 장기 실행 서버 프로세스의 에러 처리기) | 구체적인 에러와 응용 프로그램 영역에 적절한 `logging.error()`, `logging.exception()`, `logging.critical()` |

logging 함수는 이벤트의 중요도에 따라 명명된다. 아래의 표로 자세하게 설명한다. (중요도가 낮은 순으로 정렬)

| 중요도     | 사용 시기                                                    |
| ---------- | ------------------------------------------------------------ |
| `DEBUG`    | 상세한 정보, 문제를 진단해야할 경우 출력                     |
| `INFO`     | 예상대로 작동하는 지에 대한 출력                             |
| `WARNING`  | 예상치 못한 일이 발생했거나 가까운 미래에 발생할 문제에 대한 출력, 프로그램은 여전히 작동함 |
| `ERROR`    | 더욱 심각한 문제로 인하여 프로그램이 일부 기능을 수행하지 못함 |
| `CRITICAL` | 심각한 에러, 프로그램 자체가 계속 실행되지 않을 수 있음을 출력 |



- 모든 코드 예제는 `import logging`가 포함되어 있다는 가정하에 작성되었다.



### 기본 중요도 설정

logging의 기본 중요도는 `WARNING`이며, 이 중요도 이상의 이벤트만 추적되게끔 기본 설정이 되어 있는 것을 의미한다.

추적 이벤트는 여러 방식으로 처리될 수 있다. 추적 이벤트를 처리하는 가장 간단한 방법은 콘솔에 출력하는 것이며, 또 다른 일반적 방법은 디스크 파일에 기록하는 것이다.

```python
logging.warning('Watch out!') # will print a message to the console
logging.info('I told you so') # will not print anything
```

```shell
WARNING:root:Watch out!
```

기본 중요도가 `WARNING`이므로, `INFO`는 출력되지 않는다.



### 파일에 로깅

또 다른 일반적 사용법인 파일에 추적 이벤트를 출력하는 것에 대한 에제이다.

```python
logging.baseConfig(
    filename='example.log',
	encoding='utf-8',
	level=logging.DEBUG)

logging.debug('This message should go to the log file')
logging.info('So should this')
logging.warning('And this, too')
logging.error('And non-ASCII stuff, too, like Øresund and Malmö')
```

```sh
cat some_path/example.log
---
DEBUG:root:This message should go to the log file
INFO:root:So should this
WARNING:root:And this, too
ERROR:root:And non-ASCII stuff, too, like Øresund and Malmö
```

위 코드의 경우 임계값을 `DEBUG`로 설정했기 때문에 `DEBUG`보다 높은 모든 종류의 로그 중요도 메시지가 `example.log`파일에 출력되었다.



### logging 메시지 기본 포맷 변경

메시지를 표시하는 데 사용되는 포맷을 변경하려면 사용할 format을 지정해야 한다.

```python
logging.basicConfig(
    format='%(levelname):%(message)s',
    level=logging.DEBUG)

logging.debug('This message should appear on the console')
logging.info('So should this')
logging.warning('And this, too')
```

```python
#--- Console
DEBUG:This message should appear on the console
INFO:So should this
WARNING:And this, too
```

앞선 예제에서 출력된 `root`가 사라진 것을 확인할 수 있다. 하지만 간단한 사용을 위해서는 적어도 `levelname`과 `message` 그리고 `asctime`을 표시해야 할 것이다.

### logging 메시지 포맷에 날짜/시간 표시

이벤트가 발생한 날짜와 시간을 표시하려면 포맷 문자열에 `asctime`을 넣으면 된다.

```python
logging.basicConfig(format='%(asctime)s %(message)s')
logging.warning('is when this event was logged.')
```

```sh
2023-01-08 15:23:10,612 is when this event was logged.
```

위에서 출력된 날짜/시간의 기본 포맷은 `ISO8601` 또는 `RFC 3339`와 같다. 날짜/시간의 포맷을 보다 제어해야할 경우, 아래의 예제와 같이 `basicConfig`의 `datefmt`를 사용한다.

```python
logging.basicConfig(
	format='%(asctime)s %(message)s',
	datefmt='%m/%d/%Y %I:%M:%S %p')

logging.warning('is when this event was logged.')
```

```python
#--- console
01/08/2023 15:23:10 AM is when this event was logged.
```



### References

---

Python Logging Docs: https://docs.python.org/ko/3/howto/logging.html