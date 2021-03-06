<!--- Copyright (C) 2009-2015 Typesafe Inc. <http://www.typesafe.com> -->
# Configuring logging

Play uses [Logback](http://logback.qos.ch/) as its logging engine, see the [Logback documentation](http://logback.qos.ch/manual/configuration.html) for details on configuration.

## Default configuration

Play uses the following default configuration in production:

@[](/confs/play/logback-play-default.xml)

A few things to note about this configuration:

* This specifies a file appender that writes to `logs/application.log`.
* The file logger logs full exception stack traces, while the console logger only logs 10 lines of an exception stack trace.
* Play uses ANSI color codes by default in level messages.
* Play puts both the console and the file logger behind the logback [AsyncAppender](http://logback.qos.ch/manual/appenders.html#AsyncAppender).  For details on the performance implications on this, see this [blog post](https://blog.takipi.com/how-to-instantly-improve-your-java-logging-with-7-logback-tweaks/).

## Custom configuration

For any custom configuration, you will need to specify your own Logback configuration file.

### Using a configuration file from project source

You can provide a default logging configuration by providing a file `conf/logback.xml`.

### Using an external configuration file

You can also specify a configuration file via a System property.  This is particularly useful for production environments where the configuration file may be managed outside of your application source.

> Note: The logging system gives top preference to configuration files specified by system properties, secondly to files in the `conf` directory, and lastly to the default. This allows you to customize your application's logging configuration and still override it for specific environments or developer setups.

#### Using `-Dlogger.resource`

Specify a configuration file to be loaded from the classpath:

```
$ start -Dlogger.resource=prod-logger.xml
```

#### Using `-Dlogger.file`

Specify a configuration file to be loaded from the file system:

```
$ start -Dlogger.file=/opt/prod/logger.xml
```

### Examples

Here's an example of configuration that uses a rolling file appender, as well as a seperate appender for outputting an access log:

```xml
<configuration>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${user.dir}/web/logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- Daily rollover with compression -->
            <fileNamePattern>application-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 30 days worth of history -->
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date{yyyy-MM-dd HH:mm:ss ZZZZ} [%level] from %logger in %thread - %message%n%xException</pattern>
        </encoder>
    </appender>
    
    <appender name="ACCESS_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${user.dir}/web/logs/access.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover with compression -->
            <fileNamePattern>access-log-%d{yyyy-MM-dd}.gz</fileNamePattern>
            <!-- keep 1 week worth of history -->
            <maxHistory>7</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%date{yyyy-MM-dd HH:mm:ss ZZZZ} %message%n</pattern>
            <!-- this quadruples logging throughput -->
            <immediateFlush>false</immediateFlush>
        </encoder>
    </appender>

    <!-- additivity=false ensures access log data only goes to the access log -->
    <logger name="access" level="INFO" additivity="false">
        <appender-ref ref="ACCESS_FILE" />
    </logger>
    
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>

</configuration>

```

This demonstrates a few useful features:
- It uses `RollingFileAppender` which can help manage growing log files.
- It writes log files to a directory external to the application so they aren't affected by upgrades, etc.
- The `FILE` appender uses an expanded message format that can be parsed by third party log analytics providers such as Sumo Logic.
- The `access` logger is routed to a separate log file using the `ACCESS_FILE_APPENDER`.
- All loggers are set to a threshold of `INFO` which is a common choice for production logging.  

## Akka logging configuration

Akka has its own logging system which may or may not use Play's underlying logging engine depending on how it is configured.

By default, Akka will ignore Play's logging configuration and print log messages to STDOUT using its own format. You can configure the log level in `application.conf`:

```properties
akka {
  loglevel="INFO"
}
```

To direct Akka to use Play's logging engine, you'll need to do some careful configuration. First add the following config in `application.conf`:

```properties
akka {
  loggers = ["akka.event.slf4j.Slf4jLogger"]
  loglevel="DEBUG"
}
```

A couple things to note:

- Setting `akka.loggers` to `["akka.event.slf4j.Slf4jLogger"]` will cause Akka to use Play's underlying logging engine.
- The `akka.loglevel` property sets the threshold at which Akka will forward log requests to the logging engine but does not control logging output. Once the log requests are forwarded, the Logback configuration controls log levels and appenders as normal. You should set `akka.loglevel` to the lowest threshold that you will use in Logback configuration for your Akka components.
 
Next, refine your Akka logging settings in your Logback configuration:

```xml
<!-- Set logging for all Akka library classes to INFO -->
<logger name="akka" level="INFO" />
<!-- Set a specific actor to DEBUG -->
<logger name="actors.MyActor" level="DEBUG" />
```

You may also wish to configure an appender for the Akka loggers that includes useful properties such as thread and actor address.

For more information about configuring Akka's logging, including details on Logback and Slf4j integration, see the [Akka documentation](http://doc.akka.io/docs/akka/current/scala/logging.html).
