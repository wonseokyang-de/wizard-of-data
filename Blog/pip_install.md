# `pip install`

<aside> 💡 `pip install [package_name]`

</aside>

Python에는 다양한 패키지 관리 시스템이 존재합니다. 이 중 `pip`와 `conda` 가 가장 널리 사용되며 그 중 pip의 install 명령어에 대해 알아보려고 합니다.

먼저 실제 명령어가 코드로 어떻게 실행되어 서버의 패키지를 명령어의 대상이 되는 컴퓨터에 어떻게 다운받을 수 있는지에 대한 전 과정을 글로 표현하려고 합니다.

터미널에서 pip install [package_name]을 입력하면, 아래와 같은 순서로 실제 로직이 실행되는 코드의 진입점까지 실행됩니다.

실행 순서는 다음과 같습니다. (`MacOS + AnaConda`기준)

1. `pip install [package_name]`

2. `/Users/{user}/opt/anaconda3/bin/pip`

3. `/Users/{user}/opt/anaconda3/bin/pip/_internal/cli/main.py`

4. `pip/_internal/cli/main.py` - `main(args)`

5. `main(args)` - `create_command(cmd_name)`

6. `pip/_internal/commands/__init__.py` - `create_command(name, **kwargs) -> Command`

7. `create_command(name, **kwargs) -> Command` - `importlib.import_module(module_path)`

8. 명령어 입력시 전달한 `install` 을 인식

9. `pip/_internal/commands/install.py` - `InstallCommand(RequirementCommand)` 인스턴스화

10. `pip/_internal/commands/__init__.py` - `command_class(name, summary, **kwargs)`

11. `InstallCommand` 는 `__init__()` 을 가지지 않으므로, 해당 객체가 상속받은 상위 클래스인 `RequirementCommand` 의 `__init__(self, *args, **kw)` - `super().__init__(*args, **kw)` 가 실행됨

12. `RequirementCommand(IndexGroupCommand)`

13. `/Users/{user}/opt/anaconda3/bin/python`을 사용하고 있으므로, 해당 `anaconda env`의 `site-packages` 에 존재하는 `pip` 패키지 내부 폴더인 `_internal/cli/main` 의 `main()` 을 호출

    - Code

        ```python
        #!/Users/{user}/opt/anaconda3/bin/python
        
        # -*- coding: utf-8 -*-
        import re
        import sys
        
        from pip._internal.cli.main import main
        
        if __name__ == '__main__':
        		sys.argv[0] = re.sub(r'(-script\\.pyw?|\\.exe)?$', '', sys.argv[0])
        		sys.exit(main())  # 여기에서 호출됩니다.
        ```

14. `main.py - main(args)`

    - Code

        동작에 대한 부분과 관련이 적은 부분은 제거하였습니다.

        ```python
        def main(args: Optional[List[str]] = None) -> int:
            cmd_name, cmd_args = parse_command(args)  # 1
        		
        		command = create_command(cmd_name, isolated=("--isolated" in cmd_args))  # 2
        		
        		return command.main(cmd_args)  # 3
        ```

15. main

# 실제 동작이 수행되는 코드의 흐름을 따라가봅시다.

------

### 1. `cmd_name, cmd_args = parse_command(args)`

명령어를 통해 전달받은 args를 cmd_name과 cmd_args로 나누는 코드입니다. 이로 인해 cmd_name에 install이, cmd_args에는 install의 대상인 package가 들어갑니다.

### 2. `command = create_command(cmd_name, isolated=("--isolated" in cmd_args))`

전달받은 `cmd_name` 을 가지고 `site-packages/pip/_internal/commands/` 경로의 `[install.py](<http://install.py>)` 의 `InstallCommand()` 객체를 command에 반환합니다.

```python
commands_dict: Dict[str, CommandInfo] = {
    "install": CommandInfo(
        "pip._internal.commands.install",
        "InstallCommand",
        "Install packages.",
    ),
		...
}

def create_command(name: str, **kwargs: Any) -> Command:
    """
    Create an instance of the Command class with the given name.
    """
    module_path, class_name, summary = commands_dict[name]  # 2-1
    module = importlib.import_module(module_path)  # 2-2
    command_class = getattr(module, class_name)  # 2-3
    command = command_class(name=name, summary=summary, **kwargs)  # 2-4

    return command
```

### 2-1. `module_path, class_name, summary = commands_dict[name]`

### 2-2. `module = importlib.import_module(module_path)`

### 2-3. `command_class = getattr(module, class_name)  # InstallCommand`

### 2-4. `command = command_class(name=name, summary=summary, **kwargs)`

### 3. `return command.main(cmd_args)`