# Anaconda의 env에서 pip install

<aside> 💡 env에서 pip install [package] 한다면 어떤 구조 때문에 해당 env만 사용할 수 있게 되는 걸까?</aside>

```
…/opt/anaconda3/envs/env_name/lib/python/site-packages/package_name
```

- Scenario

```python
$ conda activate env_name

$ (env_name) pip install package_name
```

env를 active한 상태에서 pip를 사용하면, 해당 env에 종속된 python 내부의 pip가 사용된다. 이 pip를 통해 install한 python package들은 `.../env_name/lib/python/site-pacakges/[package]` 와 같이 저장된다.

+) 추가로 Python에서 import 사용 시 사용자가 직접 만든 util과는 다르게 install한 package의 path를 명시하지 않아도 되는 이유는 Python의 내부 폴더 중 `site-packages/` 의 경로를 자동으로 등록하여 `import` 를 바로 할 수 있다.

+) 특정 package/ 내부에 `__init__.py` 가 있으면 이 directory는 python package로 인식된다.