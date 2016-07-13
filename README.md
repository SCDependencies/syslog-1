syslog
======

[![Build Status](https://travis-ci.org/schlagert/syslog.png?branch=master)](https://travis-ci.org/schlagert/syslog)

A Syslog based logging framework for Erlang. This project is inspired by the
great work put in the two projects
[sasl_syslog](http://github.com/travelping/sasl_syslog) and
[lager](http://github.com/basho/lager). In fact `syslog` tries to combine both
approaches. In a nutshell `syslog` can be seen as a lightweight version of the
`lager` logging framework supporting only a fully compliant, Erlang-only Syslog
backend allowing remote logging.

The main difference between `sasl_syslog` and `syslog` is that `sasl_syslog`
does only provide logging of `error_logger` reports. However, the `error_logger`
is known for its bad memory consumption behaviour under heavy load (due to its
asynchronous logging mechanism). Additionally, `syslog` provides an optional
RFC 3164 (BSD Syslog) compliant protocol backend which is the only standard
supported by old versions of `syslog-ng` and `rsyslog`.

Compared to `lager`, `syslog` has a very limited set of backends. As its name
implies `syslog` is specialized in delivering its messages using Syslog only,
there is no file or console backend, no custom-written and configurable log
rotation, no line formatting and no tracing support. However, `syslog` does not
rely on port drivers or NIFs to implement the Syslog protocol and it includes
measures to enhance the overall robustness of a node, e.g. load distribution,
throughput optimization, etc.

Features
--------

* Log messages and standard `error_logger` reports according to RFC 3164
  (BSD Syslog) or RFC 5424 (Syslog Protocol) without the need for drivers,
  ports or NIFs.
* System independent logging to local or remote facilities using UDP.
* Robust event handlers - using supervised event handler subscription.
* Optionally independent error messages using a separate facility.
* Get the well-known SASL event format for `supervisor` and `crash` reports.
* Configurable verbosity of SASL printing format (printing depth is also
  configurable).
* Load distribution between all concurrently logging processes by moving the
  message formatting into the calling process(es).
* Built-in `lager` backend to bridge between both frameworks.

Planned
-------

* Configurable maximum packet size.
* Utilize the RFC 5424 _STRUCTURED-DATA_ field for `info_report`,
  `warning_report` or `error_report` with `proplists`.

Configuration
-------------

The `syslog` application already comes with sensible defaults (except for
the facilities used and the destination host). However, many things can be
customized if desired. For this purpose the following configuration options
are available and can be configured in the application environment:

* `{msg_queue_limit, Limit :: pos_integer() | infinity}`

  Specifies a limit for the number of entries allowed in the `error_logger`
  message queue. If the message queue exceeds this limit `syslog` will
  __drop the events exceeding the limit__. Default is `infinity`.

* `{protocol, rfc3164 | rfc5424}`

  Specifies which protocol standard should be used to format outgoing Syslog
  packets. Default is `rfc3164`.

* `{use_rfc5424_bom, boolean()}`

  Specifies whether the RFC5424 protocol backend should include the UTF-8 BOM
  in the message part of a Syslog packet. Default is `false`.

* `{dest_host, inet:ip_address() | inet:hostname()}`

  Specifies the host to which Syslog packets will be sent. Default is
  `{127, 0, 0, 1}`.

* `{dest_port, inet:port_number()}`

  Specifies the port to which Syslog packets will be sent. Default is `514`.

* `{facility, syslog:facility()}`

  Specifies the facility Syslog packets will be sent with. Default is `daemon`.

* `{crash_facility, syslog:facility()}`

  Specifies the facility Syslog packets with severity `crash`, will be sent
  with. It __replaces__ the previous `error_facility` property. This accompanies
  __a change in behaviour__. Starting with release `2.0.0` error messages will
  also be sent with `facility`. Only crash and supervisor reports will be sent
  to this (maybe) separate facility. If the values of the properties `facility`
  and `crash_facility` differ a short one-line summary will additionally be sent
  to `facility`. Default is `daemon`.

* `{verbose, true | {false, Depth :: pos_integer()}}`

  Configures which pretty printing mode to use when formatting `error_logger`
  reports (that is progress reports, not format messages). If verbose is
  `true` the `~p` format character will be used when formatting terms. This
  will likely result in a lot of multiline strings. If set to `{false, Depth}`
  the `~P` format character is used along with the specified printing depth.
  Default is `true`.

* `{no_progress, boolean()}`

  This flag can be used to completely omit progress reports from the log
  output. So if you you don't care when a new process is started, set this
  flag to `true`. Default is `false`.

* `{app_name, atom() | string() | binary()}`

  Configured the value reported in the `APP-NAME` field of Syslog messages. If
  not set (the default), the name part of the node name will be used. If the
  node is not alive (not running in distributed mode) the string `beam` will be
  used.

* `{log_level, syslog:severity()}`

  Configures the minimal log level. All messages with a severity value smaller
  then the configured level will be discarded. Default is `debug` (discard
  nothing).

* `{async, boolean()}`

  Specifies whether log message offloading into the `syslog_logger` event
  manager is done synchronously or asynchronously. It is highly recommended
  to leave this at its default value `false` because asynchronous delivery is
  really dangerous. A simple log burst of a few thousand processes may be enough
  to take your node down (due to out-of-memory).

If your application really needs fast asynchronous logging and you like to live
dangerously, logging can be done either with the `error_logger` or the `syslog`
API and the `syslog` application should be configured with `{async, true}`. This
sets `syslog` into asynchronous delivery mode and all message queues are
allowed to grow indefinitely.

The `syslog` application will disable the standard `error_logger` TTY output on
application startup. This has nothing to do with the standard SASL logging. It
only disables non-SASL logging via, for example `error_logger:info_msg/1,2`.
This kind of standard logging can be re-enabled at any time using the following:
```erlang
error_logger:tty(true).
```

The `syslog` application will not touch the standard SASL report handlers
attached to the `error_logger` when SASL starts. However, having SASL progress
reports on TTY can be quite annoying when trying to use the shell. The correct
way to disable this output is to configure the SASL application in the
`sys.config` of a release, for example the following line will instruct SASL
not to attach any TTY handlers to the `error_logger`:
```erlang
{sasl, [{sasl_error_logger, false}]}
```

API
---

The `syslog` application will log everything that is logged using the standard
`error_logger` API. However, __this should not be used for ordinary application
logging__.

The proper way to add logging to your application is to use the API functions
provided by the `syslog` module. These functions are similar to the ones
provided by the `error_logger` module and should feel familiar (see the
`*msg/1,2` functions).

The `syslog` application comes with a built-in, optional backend for `lager`.
This is especially useful if your release has dependencies that require `lager`
although you wish to forward logging using `syslog` only. To forward `lager`
logging into `syslog` you can use something like the following in your
`sys.config`:
```erlang
{lager, [{handlers, [{syslog_lager_backend, []}]}]}
```

Performance
-----------

TODO update on new numbers

Performance profiling has been made with a small script located in the
`benchmark` subdirectory. The figures below show the results of
`benchmark.escript all 100 10000` on an Intel(R) Core(TM)2 Duo CPU running R16B.

The above line starts a benchmark that will spawn 100 processes that each send
log message, using a specific logging framework, in a tight loop for 10000ms.
All log messages will be delivered over UDP (faked remote Syslog) to a socket
opened by the benchmark process. The total duration is the time it took to spawn
the processes, send the messages __and__ the time it took to receive all sent
messages at the socket that the benchmark process listens on.

<img src="https://cloud.githubusercontent.com/assets/404313/12110992/20d2254c-b392-11e5-83dc-64cc59bd7ad6.png" alt="benchmark results" />

As expected `syslog` and `lager` are the top performers. The main reason why
they outperform `log4erl` is the dynamic toggling of synchronous/asynchronous
logging (`log4erl` uses synchronous logging only).

Since `sasl_syslog` uses the asynchronous `error_logger` the number of messages
sent is quite huge. However, it also takes a vast amount of time and memory to
process the long `error_logger` message queue. This is also responsible for the
low number of messages sent per second in total.

A word about the performance of `lager`. Fitting `lager` into the benchmark
was unfortunately a bit tricky since the benchmark needs to know when all
messages were processed. However, `lager_syslog` uses a C port driver calling
`vsyslog` and thus does not support remote syslog. So instead of testing the
`lager_syslog_backend` the benchmark uses the `lager_console_backend`, setting
itself as the receiver for I/O messages and forwards them to the UDP socket
mentioned earlier. This would in fact slow down `lager` a bit, which would
explain the slightly better performance of `syslog`.

History
-------

### Master

* Remove dynamic switching of log message delivery. Make mode explicitly
  configurable with the new `async` directive.
* Add `app_name` configuration directive to allow configuration of the
  `APP-NAME` field value (thanks to @comtihon).
* Change severity of messages sent by `error_logger:info_[msg|report]/1,2` and
  `syslog:info_msg/1,2` from `notice` to `informational` (thanks to @comtihon).
* Add `log_level` configuration directive. With this configuration it is
  possible to discard messages with undesired severity values (thanks to
  @comtihon).
* Add optional `lager` backend to forward messages from `lager` to `syslog`.
* Further improvement of robustness, especially when many processes try to
  log concurrently by moving the message formatting away from the main event
  manager into the logging processes.

### Version 2.0.1

* Fix event handler supervision. Due to a defective match pattern in
  `syslog_monitor` crashed handlers did not get re-added as expected/supposed.

### Version 2.0.0

* Replace the property `error_facility` with `crash_facility`. Refer to the
  explanation of `crash_facility` above to learn more about this behaviour
  change.
* Performance improvements (e.g. binary as internal message format)

### Version 1.0.0

* Provide discrete API for robust `error_logger` independent logging
* Automatic toggling of sync/async logging for better load protection (default)
* Various performance improvements (e.g. timestamps, process name resolution)
* Configurable verbosity of progress report logging
* Support for release upgrades

### Version 0.0.9

* Supervised `error_logger` integration
* Message queue length based load protection
* RFC 3164 compliant backend (default)
* RFC 5424 compliant backend
* Support for local and remote facilities using UDP
* Separate facility for error messages (default off)
* Standard SASL event format for `supervisor` and `crash` reports

Supervision
-----------

<img src="https://cloud.githubusercontent.com/assets/404313/12110956/c59eec28-b391-11e5-936d-4e236f702ef0.png" alt="syslog supervision" />

For the curious; the above illustration shows the very simple supervision
hierarchy used by the `syslog` application.
