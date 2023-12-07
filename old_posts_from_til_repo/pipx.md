# pipx ensurepath를 입력했을 때 내부 동작이 궁금해요.

- 우선 `pipx` 명령어는 `저장경로(이하 …)/bin/pipx` 를 실행합니다.
- `bin/pipx` 는 심볼릭 링크로 만들어진 파일입니다. 실제 `pipx` 파일은 `…/libexec/bin/pipx` 입니다.

`…/libexec/bin/pipx` 의 내용은 아래와 같습니다.

```bash
 #!.../libexec/bin/python3.11
 # -*- coding: utf-8 -*-
 import re
 import sys
 from pipx.main import cli
 if __name__ == '__main__':
     sys.argv[0] = re.sub(r'(-script\\.pyw|\\.exe)?$', '', sys.argv[0])
     sys.exit(cli())
```

위 코드를 차례로 분석해보겠습니다.

1. `#!/opt/homebrew/Cellar/pipx/1.2.0/libexec/bin/python3.11`: 이 파일은 경로에 위치한 python3.11을 사용합니다.
2. `import re`
3. `import `sys
4. `from pipx.main import cli`
5. `if __name__ == '__main__':`
6. `sys.argv[0] = re.sub(r'(-script/.pyw|\\.exe)?$', '', sys.argv[0])`
7. `sys.exit(cli())`

이 코드에서 실제 `pipx ensurepath` 라는 기능이 동작하는 코드는 `cli()` 이며 `cli()` 는 `from pipx.main` 내부에 위치해 있습니다.

`pipx.main` 의 경로는 `.../libexec/python3.11/site-packages/pipx/main.py` 에 위치하고 있습니다.

`pipx/main.py`의 코드는 여러 내용을 포함하고 있으며, 그 중 `cli()`를 통해 `ensurepath`라는 명령어가 실행되는 코드는 다음과 같습니다.

```python
def cli() -> ExitCode:
    """Entry point from command line"""
    try:
        hide_cursor()
        parser = get_command_parser()
        argcomplete.autocomplete(parser)
        parsed_pipx_args = parser.parse_args()
        setup(parsed_pipx_args)
        check_args(parsed_pipx_args)
        if not parsed_pipx_args.command:
            parser.print_help()
            return ExitCode(1)
        return run_pipx_command(parsed_pipx_args)  # 이 때 ensurepath가 전달됩니다.
    except PipxError as e:
        print(str(e), file=sys.stderr)
        logger.debug(f"PipxError: {e}", exc_info=True)
        return ExitCode(1)
    except KeyboardInterrupt:
        return ExitCode(1)
    except Exception:
        logger.debug("Uncaught Exception:", exc_info=True)
        raise
    finally:
        logger.debug("pipx finished.")
        show_cursor()
        
        
def run_pipx_command(args: argparse.Namespace) -> ExitCode:  # noqa: C901
    verbose = args.verbose if "verbose" in args else False
    pip_args = get_pip_args(vars(args))
    venv_args = get_venv_args(vars(args))
    venv_container = VenvContainer(constants.PIPX_LOCAL_VENVS)

    if "package" in args:
        package = args.package
        if urllib.parse.urlparse(package).scheme:
            raise PipxError("Package cannot be a url")
        if "spec" in args and args.spec is not None:
            if urllib.parse.urlparse(args.spec).scheme:

    if "package" in args:
        package = args.package
        if urllib.parse.urlparse(package).scheme:
            raise PipxError("Package cannot be a url")
        if "spec" in args and args.spec is not None:
            if urllib.parse.urlparse(args.spec).scheme:
                if "#egg=" not in args.spec:
                    args.spec = args.spec + f"#egg={package}"
        venv_dir = venv_container.get_venv_dir(package)
        logger.info(f"Virtual Environment location is {venv_dir}")

    if "skip" in args:
       skip_list = [canonicalize_name(x) for x in args.skip]

    if args.command == "run":
				# ...

		elif args.command == "ensurepath":  # 여기에서 분기를 타고 실행됩니다.
		    try:
		        return commands.ensure_pipx_paths(force=args.force)
		    except Exception as e:
		        logger.debug("Uncaught Exception:", exc_info=True)
		        raise PipxError(str(e), wrap_message=False)
```

elif 분을 타고 아래로 내려가면 `commands.ensure_pipx_paths(force=args.force)` 라는 코드가 실행됩니다. 실제 동작되는 로직을 보려면 commands를 import한 `from pipx import commands, constants` 중 `commands/`로 이동해야 합니다.

commands는 python 파일이 아닌 directory 였습니다. 이는 이 directory 내부에 `__init__.py` 라는 파일이 존재하여 directory를 python package로 인식하게끔 처리한 것입니다. `commands/` 의 내용은 아래와 같습니다.

```python
from pipx.commands.ensure_path import ensure_pipx_paths  # 드디어 내부 로직이 포함된 코드를 찾았습니다!
from pipx.commands.environment import environment
from pipx.commands.inject import inject
from pipx.commands.install import install
from pipx.commands.list_packages import list_packages
from pipx.commands.reinstall import reinstall, reinstall_all
from pipx.commands.run import run
from pipx.commands.run_pip import run_pip
from pipx.commands.uninject import uninject
from pipx.commands.uninstall import uninstall, uninstall_all
from pipx.commands.upgrade import upgrade, upgrade_all

__all__ = [
    "upgrade",
    "upgrade_all",
    "run",
    "install",
    "inject",
    "uninject",
    "uninstall",
    "uninstall_all",
    "reinstall",
    "reinstall_all",
    "list_packages",
    "run_pip",
    "ensure_pipx_paths",
    "environment",
]
```

------

이 부분부터는 실제 ensurepath의 서비스 로직입니다. 추후 아래의 내용도 분석해볼 수 있으면 좋겠습니다.

`from pipx.commands.ensure_path import ensure_pipx_paths` 의 내용은 아래와 같습니다.

```python
import logging
import site
import sys
from pathlib import path
from typing import optional, tuple

import userpath  # type: ignore

from pipx import constants
from pipx.constants import exit_code_ok, exitcode
from pipx.emojis import hazard, stars
from pipx.util import pipx_wrap

logger = logging.getlogger(__name__)

def get_pipx_user_bin_path() -> optional[path]:
    """returns none if pipx is not installed using `pip --user`
    otherwise returns parent dir of pipx binary
    """
    # note: using this method to detect pip user-installed pipx will return
    #   none if pipx was installed as editable using `pip install --user -e`

    # <https://docs.python.org/3/install/index.html#inst-alt-install-user>
    #   linux + mac:
    #       scripts in <userbase>/bin
    #   windows:
    #       scripts in <userbase>/python<xy>/scripts
    #       modules in <userbase>/python<xy>/site-packages

    pipx_bin_path = none

    script_path = path(__file__).resolve()
    userbase_path = path(site.getuserbase()).resolve()
    try:
        _ = script_path.relative_to(userbase_path)
    except valueerror:
        pip_user_installed = false
    else:
        pip_user_installed = true
    if pip_user_installed:
        test_paths = (
            userbase_path / "bin" / "pipx",
            path(site.getusersitepackages()).resolve().parent / "scripts" / "pipx.exe",
        )
        for test_path in test_paths:
            if test_path.exists():
                pipx_bin_path = test_path.parent
                break

    return pipx_bin_path

def ensure_path(location: path, *, force: bool) -> tuple[bool, bool]:
    """ensure location is in user's path or add it to path.
    returns true if location was added to path
    """
    location_str = str(location)
    path_added = false
    need_shell_restart = userpath.need_shell_restart(location_str)
    in_current_path = userpath.in_current_path(location_str)

    if force or (not in_current_path and not need_shell_restart):
        userpath.append(location_str, "pipx")
        print(
            pipx_wrap(
                f"success! added {location_str} to the path environment variable.",
                subsequent_indent=" " * 4,
            )
        )
        path_added = true
        need_shell_restart = userpath.need_shell_restart(location_str)
    elif not in_current_path and need_shell_restart:
        print(
            pipx_wrap(
                f"""
                {location_str} has been been added to path, but you need to
                open a new terminal or re-login for this path change to take
                effect.
                """,
                subsequent_indent=" " * 4,
            )
        )
    else:
        print(
            pipx_wrap(f"{location_str} is already in path.", subsequent_indent=" " * 4)
        )

    return (path_added, need_shell_restart)

def ensure_pipx_paths(force: bool) -> exitcode:
    """returns pipx exit code."""
    bin_paths = {constants.local_bin_dir}

    pipx_user_bin_path = get_pipx_user_bin_path()
    if pipx_user_bin_path is not none:
        bin_paths.add(pipx_user_bin_path)

    path_added = false
    need_shell_restart = false
    for bin_path in bin_paths:
        (path_added_current, need_shell_restart_current) = ensure_path(
            bin_path, force=force
        )
        path_added |= path_added_current
        need_shell_restart |= need_shell_restart_current

    print()

    if path_added:
        print(
            pipx_wrap(
                """
                consider adding shell completions for pipx. run 'pipx
                completions' for instructions.
                """
            )
            + "\\n"
        )
    elif not need_shell_restart:
        sys.stdout.flush()
        logger.warning(
            pipx_wrap(
                f"""
                {hazard}  all pipx binary directories have been added to path. if you
                are sure you want to proceed, try again with the '--force'
                flag.
                """
            )
            + "\\n"
        )

    if need_shell_restart:
        print(
            pipx_wrap(
                """
                you will need to open a new terminal or re-login for the path
                changes to take effect.
                """
            )
            + "\\n"
        )

    print(f"otherwise pipx is ready to go! {stars}")

    r
```