---
title: "mysqlclinet 설치 오류" #Article title.
date: 2023-07-16
category: [ERROR, PYTHON] #One, more categories or no at all.
tag: [error,pymysql]
---

- error log

```
Collecting mysqlclient
Using cached mysqlclient-2.2.0.tar.gz (89 kB)
Installing build dependencies ... done
Getting requirements to build wheel ... error
error: subprocess-exited-with-error × Getting requirements to build wheel did not run successfully.
│ exit code: 1
╰─> [24 lines of output]
Trying pkg-config --exists mysqlclient
/bin/sh: 1: pkg-config: not found
Command 'pkg-config --exists mysqlclient' returned non-zero exit status 127.
Trying pkg-config --exists mariadb
/bin/sh: 1: pkg-config: not found
Command 'pkg-config --exists mariadb' returned non-zero exit status 127.
Traceback (most recent call last):
File "/app/venv/lib/python3.9/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 353, in <module>
main()
File "/app/venv/lib/python3.9/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 335, in main
json_out['return_val'] = hook(**hook_input['kwargs'])
File "/app/venv/lib/python3.9/site-packages/pip/_vendor/pyproject_hooks/_in_process/_in_process.py", line 118, in get_requires_for_build_wheel
return hook(config_settings)
File "/tmp/pip-build-env-4o8v4dtt/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 341, in get_requires_for_build_wheel
return self._get_build_requires(config_settings, requirements=['wheel'])
File "/tmp/pip-build-env-4o8v4dtt/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 323, in _get_build_requires
self.run_setup()
File "/tmp/pip-build-env-4o8v4dtt/overlay/lib/python3.9/site-packages/setuptools/build_meta.py", line 338, in run_setup
exec(code, locals())
File "<string>", line 154, in <module>
File "<string>", line 48, in get_config_posix
File "<string>", line 27, in find_package_name
Exception: Can not find valid pkg-config name.
Specify MYSQLCLIENT_CFLAGS and MYSQLCLIENT_LDFLAGS env vars manually
[end of output] note: This error originates from a subprocess, and is likely not a problem with pip.
error: subprocess-exited-with-error × Getting requirements to build wheel did not run successfully.
│ exit code: 1
╰─> See above for output. note: This error originates from a subprocess, and is likely not a problem with pip
```

`pkg-config` 가 없어서 생기는 에러

```python
#1
apt-get update

#2
apt-get install -y pkg-config

#3
apt-get install -y libmariadb-dev

#4
pip install mysqlclient
```

위의 코드를 순서대로 실행해도 안되는 경우가 있음

**`error: command 'gcc' failed: No such file or directory`**

요런 에러 로그

```python
apt-get install -y build-essential
```

얘도 install 해주면 잘됨!

아키텍처 문제인 경우가 대다수


- 참고

1. https://stackoverflow.com/questions/74730749/how-to-install-mysqlclient-python-library-in-linux

2. https://askubuntu.com/questions/1219870/error-command-x86-64-linux-gnu-gcc-failed-with-exit-status-1-for-lssl-and-lc