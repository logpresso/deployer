# Logpresso Deployer

We developed Logpresso Deployer to manage infrastructure as code, like containers, even in bare metal environments.

## Download

* [Logpresso Deployer 1.0.2205.0 (Windows x64)](https://github.com/logpresso/deployer/releases/download/1.0.2305.0/logpresso-deployer-1.0.2205.0.zip)
* [Logpresso Deployer 1.0.2205.0 (Linux x64)](https://github.com/logpresso/deployer/releases/download/1.0.2305.0/logpresso-deployer-1.0.2305.0.tar.gz)
* [Logpresso Deployer 1.0.2205.0 (Any OS)](https://github.com/logpresso/deployer/releases/download/1.0.2305.0/logpresso-deployer-1.0.2205.0.jar)

## Usage

Deployer runs the deployment commands according to the DSL (Domain Specific Language) in the script.

```
Logpresso Deployer 1.0.2305.0 (build 20230507)

Usage: deployer.jar [script path] [arg1] [arg2] ..
```

### Getting Started

The simplest use case is to run commands remotely over SSH.

```
connect host:port account password
exec uname -a
disconnect
```

To upload and unzip the Logpresso package file, follow these steps:

```
connect HOST:PORT ACCOUNT PASSWORD
upload D:\files\logpresso-MAE-4.0.2303.0-u3009.zip test.zip
exec unzip test.zip
disconnect
```

IPs, accounts, passwords, file paths, etc. are used repeatedly, so variables should be introduced to reuse them. Variables can be defined with the `set` command, and variable references are in the form `${name}`. Execution arguments are passed in the order `${0}`, `${1}` .... If a line starts with `#`, it is recognized as a comment. For example

```
set var1 hello
set var2 world
set var3 ${var1} ${var2}
# Check result
echo run argument ${0}
echo ${var1} + ${var2} = ${var3}
```

To access the Logpresso shell to perform a command and save the result to a variable, do the following:

```
connect HOST:PORT ACCOUNT PASSWORD
logpresso --password logpresso --cmd uptime --var up
echo ${up}
disconnect
```

### Commands

#### set

```
set NAME VALUE
```

- VALUE can be any number of string tokens, including spaces.
- When the script engine tries to run the next line of the script, it replaces all `${NAME}` for the variables set so far.

#### connect

```
connect HOST:PORT USER PASSWORD
```

- If you omit the `:PORT` part, you will connect to port 22.
- You cannot connect to more than one SSH server at a time. To connect to another SSH server, you must command `disconnect`.
- Most commands assume that you are connected to SSH, so after the `set` command section, you will typically start with the `connect` command.

#### disconnect

```
disconnect
```

- Terminates the current connection to the SSH server.

#### upload

```
upload LOCAL_PATH REMOTE_PATH
```

- Uploads a file from the local path to the remote path.
- Typically, you would upload the file to the home directory of a regular account registered with sudoers first, and then use sudo for the `exec` command to move the file to the desired directory.

#### download

```
download REMOTE_PATH LOCAL_PATH
```

- Download a file from a remote path to a local path.
- Parameters are described in the same order as upload, SRC → DST.

#### echo

```
echo ${NAME}
```

- Replaces a variable and prints it to the console. This is often used for debugging scripts.

#### exec

```
exec COMMAND
```

- Enter a command to run remotely.
- Be careful not to run commands that require user input, as this will cause the script to hang while running.
    - For example, if you use the `unzip` command to unpack a file, you may be asked if you want to overwrite a file that already exists. In this case, the deployer will wait indefinitely.
    - In this case, change the command to avoid user input by adjusting switches, such as adding the `-o` option to force an overwrite.
- The old version of the run.sh script, `sudo -u logpresso /opt/logpresso/run.sh start|stop`, did not have permission to the current directory when executed, causing the find command to locate ARAQNE_CORE_VERSION with the error `(error retrieving current directory: getcwd: cannot access parent directories: No such file or directory)` error may occur. Add `cd $PKGDIR` immediately after `PKGDIR="$(dirname "$RUN")"` in run.sh.

#### setcap

```
setcap JAVA_HOME_PATH
```

- This command adds the JAVA_HOME_PATH path to /etc/ld.so.conf.d/java_jemalloc.conf.
    - Execute `sudo echo JAVA_HOME_PATH/lib/jli' | sudo tee /etc/ld.so.conf.d/java_jemalloc.conf`.
- Then run `sudo setcap cap_sys_time,cap_net_bind_service=+ep JAVA_HOME_PATH/bin/java`
- Then run `sudo ldconfig`
- Note that setcap and ldconfig are run even if the java_jemalloc.conf file exists. This is to ensure that setcap is set at the time these commands are executed, as chown and the like can sometimes reset the permissions of the java executable.

#### set-config

```
set-config --template TEMPLATE_PATH --output OUTPUT_PATH --java_home JAVA_HOME --xms 2G --xmx 4G --dmem 4G --data_dir DATA_PATH
```

- Reads the server's configuration template file, changes the configuration values, and writes them to the output location on the specified server.
- (Required) template: config.def.sh template file path
- (Required) output: config.sh file path
- (required) java_home: JDK path. JAVA_HOME entry in config.sh
- (optional) xms: Minimum heap size. MIN_HEAP_SIZE entry in config.sh
- (optional) xmx: Maximum heap size. MAX_HEAP_SIZE entry in config.sh
- (Optional) dmem: Direct memory size. MAX_DIRECT_MEM_SIZE entry in config.sh
- (optional) data_dir: Data directory path. DATADIR entry in config.sh

#### wait-http

```
wait-http URL TIMEOUT
```

- Wait for the specified URL to respond with a `200` status.
- This is used to wait for the web console to become operational and REST API serviceable after the Logpresso daemon boots.
    - Depending on the timing of the bundle boot, you might need to specify the URL finely.
    - For example, even with the web server index loaded, the installer REST API might not be ready.
    - Therefore, specify exactly which resources in the bundle you want to wait for.
- TIMEOUT: Wait time in seconds

#### wait-port

```
wait-port PORT TIMEOUT
```

- Wait for the specified TCP port to be opened.
    - Wait for the results of `sudo netstat -na | grep LISTEN | grep tcp | grep :port` to be output
- This is used to wait for an SSH port or Telnet port to open after the Logpresso daemon boots.

#### logpresso

```
logpresso --password PASSWORD --default_password DEFAULT_PASSWORD --cmd COMMAND --var VAR --expects EXPECT1 >> INPUT1 >> EXPECT2 >> INPUT2 ...
```

- Connect to the Logpresso shell, run the command, and set the result to a variable.
- (Required) password: Password for the Logpresso shell `root` account
- (Optional) default_password: Specify a default password if this is the first connection. If the default password option is specified, change to the `password` value on first connection and start.
- (Required) cmd: Logpresso shell command
- (Optional) var: Variable name to store the command execution results in
- (Optional) expects: If multiple prompts are issued when running cmd, enter the prompt string you expect and the corresponding input string in order, separated by `>>`.

For example, you can run the `logpresso.createJdbcProfile` command as shown below:

```
logpresso --password SSH_PASSWORD --cmd logpresso.createJdbcProfile sonar --expects connection string? >> ${conn_string} >> read only? >> >> user? >> sonar >> password? >> ${db_password} >> confirm password? >> ${db_password} >> granted users (csv)? >> >> granted groups (csv)? >> >>
```

#### install

```
install --url URL --locale en --company_name COMPANY --user_name USER_NAME --login_name LOGIN_NAME --password PASSWORD --email EMAIL --web_endpoint WEB_ENDPOINT --db_host DB_HOST --db_port 3306 --db_user sonar --db_user sonar --db_password DB_PASSWORD --timeout 120
```

Run the web installer of Logpresso Sonar remotely.
- (Required) URL: `https://hostname:port` Format. Deployer emits `POST https://hostname:port/install/`
- (Required) locale: `en` or `ko`
- (Optional) isac_key: ISAC API key
- (Required) company_name: Company name
- (Required) user_name: Cluster administrator's name
- (Required) login_name: Cluster administrator's account
- (Required) password: Cluster administrator's password
- (Required) email: Cluster administrator's email address
- (Required) web_endpoint: Access address included when sending a call to action. Must start with `https://`
- (Required) db_host: Database address. On-premises environments typically specify `localhost`.
- (Optional) db_port: Default is `3306`.
- (Required) db_name: Database name. On-premises environments typically specify `sonar`.
- (Required) db_user: Database account. Typically specifies `sonar`.
- (Required) db_password: Database password
- (Optional) timeout: default `120` seconds

For example, you can run it like this:

```
install --url ${install_url} --locale ${locale} --company_name ${company_name} --user_name ${user_name} --login_name ${login_name} --password ${user_password} --email ${email} --db_host ${db_host} --db_name sonar --db_user sonar --db_password ${db_password} --web_endpoint ${web_endpoint} --timeout ${install_timeout}
```

#### rex

```
rex REGEX
```

- The `exec` and `logpresso` commands store their output in the `${_}` variable after execution.
- The rex command extracts the variable by applying a regular expression to the `${_}` variable.

For example, you can extract the guid value from the JSON response after running the curl command, as shown below.

```
exec curl -ks -H "Authorization: Bearer ${api_key}" https://localhost/api/sonar/node-pairs?include_nodes=true
rex "guid1":"(?<c1_guid>[^\"]+)"
```

#### run

```
run SCRIPT_PATH ARG1 ARG2 ...
```

- Run another modularized script file.
- This is often used when you want to run the same task repeatedly, changing only the IP for each cluster node.

## Contact

If you are a Logpresso engineer and have an issue, please open a case at https://support.logpresso.com.
