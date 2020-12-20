### Configuration Filters

[Configuration Filters](https://www.bookstack.cn/read/FlumeUserGuide-1.9-en/spilt.4.flume.md)

Flume provides a tool for injecting sensitive or generated data into the configuration in the form of configuration filters. A configuration key can be set as the value of configuration propertiesand it will be replaced by the configuration filter with the value it represents.

The format is similar to the Java Expression Language, howeverit is currently not a fully working EL expression parser, just a format that looks like it.

```ini
<agent_name>.configfilters = <filter_name>
<agent_name>.configfilters.<filter_name>.type = <filter_type>
 
<agent_name>.sources.<source_name>.parameter = ${<filter_name>['<key_for_sensitive_or_generated_data>']}
<agent_name>.sinks.<sink_name>.parameter = ${<filter_name>['<key_for_sensitive_or_generated_data>']}
```

There are 3 kinds of configuration filters: 

- ENV: org.apache.flume.configfilter.EnvironmentVariableConfigFilter
- EXTERNAL: org.apache.flume.configfilter.ExternalProcessConfigFilter
- HADOOP: org.apache.flume.configfilter.HadoopCredentialStoreConfigFilter

Also, you can customize your configuration filters by implementing `ConfigFilter`.

### EnvironmentVariableConfigFilter

| Property Name |	Default |	Description |
| --- | --- | --- |
| type |	–	| The component type name has to be `env` |

**Example:**

```ini
a1.sources = r1
a1.channels = c1
a1.configfilters = f1
 
a1.configfilters.f1.type = env
 
a1.sources.r1.channels =  c1
a1.sources.r1.type = http
a1.sources.r1.keystorePassword = ${f1['my_keystore_password']} #will get the value Secret123
```

Here the `a1.sources.r1.keystorePassword` configuration property will get the value of the **my_keystore_password** environment variable. One way to set the environment variable is to run flume agent like this:

```shell
$ my_keystore_password=Secret123 bin/flume-ng agent —conf conf —conf-file example.conf …
```

### ExternalProcessConfigFilter

| Property Name |	Default |	Description |
| --- | --- | --- |
| type	| –	| The component type name has to be `external` | 
| command	| –	| The command that will be executed to get the value for the given key. The command will be called like: <command> <key> And expected to return a single line value with exit code 0. | 
| charset	| UTF-8	| The characterset of the returned string. | 

**Example:**

To hide a password in the configuration set its value as in the following example.

```ini
a1.sources = r1
a1.channels = c1
a1.configfilters = f1
 
a1.configfilters.f1.type = external
a1.configfilters.f1.command = /usr/bin/passwordResolver.sh
a1.configfilters.f1.charset = UTF-8
 
a1.sources.r1.channels =  c1
a1.sources.r1.type = http
a1.sources.r1.keystorePassword = ${f1['my_keystore_password']} #will get the value Secret123
```

In this example flume will run the following command to get the value

```bash
$ /usr/bin/passwordResolver.sh my_keystore_password
```

The `passwordResolver.sh` will return Secret123 with an exit code 0.

### HadoopCredentialStoreConfigFilter

A hadoop-common library needed on the classpath for this feature (2.6+ version).If hadoop is installed the agent adds it to the classpath automatically.


| Property Name |	Default |	Description |
| --- | --- | --- |
| type	| –	| The component type name has to be `hadoop` | 
| credential.provider.path	| –	| The provider path. See hadoop documentation _here: https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/CredentialProviderAPI.html#Configuring_the_Provider_Path | 
| credstore.java-keystore-provider.password-file	| –	| The name of the password file if a file is used to store the password. The file must e on the classpath.Provider password can be set with the HADOOP_CREDSTORE_PASSWORD environment variable or left empty. | 

**Example:**

To hide a password in the configuration set its value as in the following example.

```ini
a1.sources = r1
a1.channels = c1
a1.configfilters = f1

a1.configfilters.f1.type = hadoop
a1.configfilters.f1.credential.provider.path = jceks://file/<path_to_jceks file>

a1.sources.r1.channels = c1
a1.sources.r1.type = http
a1.sources.r1.keystorePassword = ${f1['my_keystore_password']} #will get the value from the credential store
```
