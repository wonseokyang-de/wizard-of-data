# ERROR: Could not find a version that satisfies the requirement scann



scann을 다운받으면, 다음과 같은 에러가 터미널에 출력된다.

<aside> 🚨 **ERROR: Could not find a version that satisfies the requirement scann (from versions: none) ERROR: No matching distribution found for scann**

</aside>

위 에러가 어떤 상황일 때 발생하는지 궁금하여 pip의 내부 코드를 살펴보니 아래와 같은 코드를 발견하였다.

```python
if installed_version is None and best_candidate is None:
		logger.critical(
				"Could not find a version that satisfies the requirement %s "
				"(from versions: %s)",
				req,
				_format_versions(best_candidate_result.iter_all()),
		)

    raise DistributionNotFound(
        "No matching distribution found for {}".format(req)
    )
```

위 에러를 발생시키는 조건은 코드의 첫 번째 라인과 같이 `installed_version` 과 `best_candidate` 둘 다 `None` 일 경우 이다.

각 조건에 필요한 요소는 다음과 같다.

- `installed_version` :

    말 그대로 현재 설치되어 있는 버전(?)

    - 세부 코드 동작 과정

        `installed_version` 이라는 변수가 선언되는 과정은 아래의 코드와 같다.

        ```python
        installed_version: Optional[_BaseVersion] = None
        
        if req.satisfied_by is not None:
            installed_version = req.satisfied_by.version
        ```

        코드를 확인해보면 `installed_version` 은 `req` 라는 객체를 사용하여 선언되므로 먼저 이 객체에 대해 알아볼 필요가 있다.

        `Resolver.resolve(root_reqs: List[InstallRequirement])` → `…` → `find_requirement()`

- `best_candidate` :

    최신인 것 중 stable한 패키지를 찾는 것 같다(?)

```python
find_all_candidates() -> find_best_candidate() -> best_candidate
```