# 简介

## wait-for-it

[wait-for-it 官网](https://github.com/vishnubob/wait-for-it 'wait-for-it 官网')

### Usage

```
wait-for-it.sh host:port [-s] [-t timeout] [-- command args]
-h HOST | --host=HOST       Host or IP under test
-p PORT | --port=PORT       TCP port under test
                            Alternatively, you specify the host and port as host:port
-s | --strict               Only execute subcommand if the test succeeds
-q | --quiet                Don't output any status messages
-t TIMEOUT | --timeout=TIMEOUT
                            Timeout in seconds, zero for no timeout
-- COMMAND ARGS             Execute command with args after the test finishes
```

### Examples

```bash
# ./wait-for-it.sh www.google.com:80 -- echo "google is up"
wait-for-it.sh: waiting 15 seconds for www.google.com:80
wait-for-it.sh: www.google.com:80 is available after 0 seconds
google is up

# ./wait-for-it.sh -t 0 www.google.com:80 -- echo "google is up"
wait-for-it.sh: waiting for www.google.com:80 without a timeout
wait-for-it.sh: www.google.com:80 is available after 0 seconds
google is up

# ./wait-for-it.sh www.google.com:81 --timeout=1 --strict -- echo "google is up"
wait-for-it.sh: waiting 1 seconds for www.google.com:81
wait-for-it.sh: timeout occurred after waiting 1 seconds for www.google.com:81
wait-for-it.sh: strict mode, refusing to execute subprocess

# ./wait-for-it.sh www.google.com:80
wait-for-it.sh: waiting 15 seconds for www.google.com:80
wait-for-it.sh: www.google.com:80 is available after 0 seconds

# echo $?
0

# ./wait-for-it.sh www.google.com:81
wait-for-it.sh: waiting 15 seconds for www.google.com:81
wait-for-it.sh: timeout occurred after waiting 15 seconds for www.google.com:81

# echo $?
124
```

## wait-for

[wait-for 官网](https://github.com/Eficode/wait-for 'wait-for 官网')

### Usage

```
./wait-for host:port|url [-t timeout] [-- command args]
  -q | --quiet                        Do not output any status messages
  -t TIMEOUT | --timeout=timeout      Timeout in seconds, zero for no timeout
  -v | --version                      Show the version of this tool
  -- COMMAND ARGS                     Execute command with args after the test finishes
```

### Examples

```bash
# ./wait-for www.eficode.com:80 -- echo "Eficode site is up"
Eficode site is up

# ./wait-for https://www.eficode.com -- echo "Eficode is accessible over HTTPS"
Eficode is accessible over HTTPS
```

***注: 在 docker 容器里使用 ```wait-for```，需要安装 ```nc```，默认位置在 ```/usr/bin/nc```；否则会报错 ```nc command is missing```。***
