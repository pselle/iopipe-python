# IOpipe Analytics & Distributed Tracing Agent for Python

[![PyPI version](https://badge.fury.io/py/iopipe.svg)](https://badge.fury.io/py/iopipe)
[![Slack](https://img.shields.io/badge/chat-slack-ff69b4.svg)](https://iopipe.now.sh)

This package provides analytics and distributed tracing for event-driven applications running on AWS Lambda.

- [Installation](#installation)
- [Usage](#usage)
  - [Configuration](#configuration)
  - [Reporting Exceptions](#reporting-exceptions)
  - [Custom Metrics](#custom-metrics)
- [Plugins](#plugins)
  - [Event Info Plugin](#event-info-plugin)
  - [Logging Plugin](#logging-plugin)
  - [Profiler Plugin](#profiler-plugin)
  - [Trace Plugin](#trace-plugin)
  - [Creating Plugins](#creating-plugins)
- [Supported Python Versions](#supported-python-versions)
- [Framework Integration](#framework-integration)
  - [Chalice](#chalice)
  - [Zappa](#zappa)
- [License](#license)

## Installation

We expect you will import this library into an existing (or new) Python project intended to be run on AWS Lambda.  On Lambda, functions are expected to include module dependencies within their project paths, thus we use `-t $PWD`. Users building projects with a requirements.txt file may simply add `iopipe` to their dependencies.

From your project directory:

```bash
$ pip install iopipe -t .

# If running locally or in other environments _besides_ AWS Lambda:
$ pip install requests -t .
```

Your folder structure for the function should look similar to:

```
index.py # contains your lambda handler
  /iopipe
    - __init__.py
    - iopipe.py
  /requests
    - __init__.py
    - api.py
    - ...
```

Installation of the requests library is necessary for local dev/test, but not when running on AWS Lambda as this library is part of the default environment via the botocore library.

More details about lambda deployments are available in the [AWS documentation](https://docs.aws.amazon.com/lambda/latest/dg/lambda-python-how-to-create-deployment-package.html).

## Usage

Simply use our decorator to report metrics:

```python
from iopipe import IOpipe

iopipe = IOpipe('your project token here')

@iopipe
def handler(event, context):
  pass
```

### Configuration

The following may be set as kwargs to the IOpipe class initializer:

#### `token` (string: required)

Your IOpipe project token. If not supplied, the environment variable `$IOPIPE_TOKEN` will be used if present. [Find your project token](https://dashboard.iopipe.com/install)

#### `debug` (bool: optional = False)

Debug mode will log all data sent to IOpipe servers. This is also a good way to evaluate the sort of data that IOpipe is receiving from your application. If not supplied, the environment variable `IOPIPE_DEBUG` will be used if present.

#### `enabled` (bool: optional = True)

Conditionally enable/disable the agent. For example, you will likely want to disabled the agent during development. The environment variable `$IOPIPE_ENABLED` will also be checked.

#### `network_timeout` (int: optional = 5000)

The number of milliseconds IOpipe will wait while sending a report before timing out. If not supplied, the environment variable `$IOPIPE_NETWORK_TIMEOUT` will be used if present.

#### `timeout_window` (int: optional = 500)

By default, IOpipe will capture timeouts by exiting your function 500 milliseconds early from the AWS configured timeout, to allow time for reporting. You can disable this feature by setting `timeout_window` to `0` in your configuration. If not supplied, the environment variable `$IOPIPE_TIMEOUT_WINDOW` will be used if present.

### Reporting Exceptions

The IOpipe decorator will automatically catch, trace and reraise any uncaught exceptions in your function. If you want to trace exceptions raised in your case, you can use the `.error(exception)` method. This will add the exception to the current report.

```python
from iopipe import IOpipe

iopipe = IOpipe()

# Example 1: uncaught exceptions

@iopipe
def handler(event, context):
  raise Exception('This exception will be added to the IOpipe report automatically')

# Example 2: caught exceptions

@iopipe
def handler(event, context):
  try:
    raise Exception('This exception is being caught by your function')
  except Exception as e:
    # Makes sure the exception is added to the report
    context.iopipe.error(e)
```

It is important to note that a report is sent to IOpipe when `error()` is called. So you should only record exceptions this way for failure states. For caught exceptions that are not a failure state, it is recommended to use custom metrics (see below).

### Custom Metrics

You can log custom values in the data sent upstream to IOpipe using the following syntax:

```python
from iopipe import IOpipe

iopipe = IOpipe()

@iopipe
def handler(event, context):
  # the name of the metric must be a string
  # numerical (int, long, float) and string types supported for values
  context.iopipe.metric('my_metric', 42)
```

Metric key names are limited to 128 characters, and string values are limited to 1024 characters.

### Label

Label invocations sent to IOpipe by calling the `label` method with a string (limit of 128 characters):

```python
from iopipe import IOpipe

iopipe = IOpipe()

@iopipe
def handler(event, context):
  # the name of the tag must be a string
  context.iopipe.label('this-invocation-is-special')
```

## Plugins

IOpipe's functionality can be extended through plugins. Plugins hook into the agent lifecycle to allow you to perform additional analytics.

### Event Info Plugin

The IOpipe agent comes bundled with an event info plugin that automatically extracts useful information from the `event`
object and creates custom metrics for them.

Here's an example of how to use the event info plugin:

```python
from iopipe import IOpipe
from iopipe.contrib.eventinfo import EventInfoPlugin

iopipe = IOpipe(plugins=[EventInfoPlugin()])

@iopipe
def handler(event, context):
    # do something here
```

When this plugin is installed, custom metrics will be created automatically for the following event source data:

* API Gateway
* CloudFront
* Kinesis
* Kinesis Firehose
* S3
* SES
* SNS
* Scheduled Events

### Logging Plugin

The IOpipe agent comes bundled with a logging plugin that allows you to attach IOpipe to the `logging` module so that you can see your log messages in the IOpipe dashboard.

Here's an example of how to use the logging plugin:

```python
from iopipe import IOpipe
from iopipe.contrib.logging import LoggingPlugin

iopipe = IOpipe(plugins=[LoggingPlugin()])

@iopipe
def handler(event, context):
    context.iopipe.log.info('Handler has started execution')
```

Since this plugin just adds a handler to the `logging` module, you can use `logging` directly as well:

```python
import logging

from iopipe import IOpipe
from iopipe.contrib.logging import LoggingPlugin

iopipe = IOpipe(plugins=[LoggingPlugin()])
logger = logging.getLogger()

@iopipe
def handler(event, context):
    logger.error('Uh oh')
```

You can also specify a log name, such as if you only wanted to log messages for `mymodule`:

```python
from iopipe import IOpipe
from iopipe.contrib.logging import LoggingPlugin

iopipe = IOpipe(plugins=[LoggingPlugin('mymodule')])
```

This would be equivalent to `logging.getLogger('mymodule')`.

By default, the logging plugin log level is `logging.INFO`, but it can be set like this:

```python
import logging

from iopipe import IOpipe
from iopipe.contrib.logging import LoggingPlugin

iopipe = IOpipe(plugins=[LoggingPlugin(level=logging.DEBUG)])
```

Putting IOpipe into `debug` mode also sets the log level to `logging.DEBUG`.

The logging plugin also redirects stdout by default, so you can do the following:

```python
from iopipe import IOpipe
from iopipe.contrib.logging import LoggingPlugin

iopipe = IOpipe(plugins=[LoggingPlugin()])

@iopipe
def handler(event, context):
    print('I will be logged')
```

If you prefer your print statements not to be logged, you can disable this by setting `redirect_stdout` to `False`:

```python
iopipe = IOpipe(plugins=[LoggingPlugin(redirect_stdout=False)])
```

### Profiler Plugin

The IOpipe agent comes bundled with a profiler plugin that allows you to profile your functions with [cProfile](https://docs.python.org/3/library/profile.html#module-cProfile).

Here's an example of how to use the profiler plugin:

```python
from iopipe import IOpipe
from iopipe.contrib.profiler import ProfilerPlugin

iopipe = IOpipe(plugins=[ProfilerPlugin()])

@iopipe
def handler(event, context):
    # do something here
```

By default the plugin will be disabled and can be enabled at runtime by setting the `IOPIPE_PROFILER_ENABLED` environment variable to `true`/`True`.

If you want to enable the plugin for all invocations:

```python
iopipe = IOpipe(plugins=[ProfilerPlugin(enabled=True)])
```

Now in your IOpipe invocation view you will see a "Profiling" section where you can download your profiling report.

Once you download the report you can open it using pstat's interactive browser with this command:

```bash
python -m pstats <file here>
```

Within the pstats browser you can sort and restrict the report in a number of ways, enter the `help` command for details. Refer to the [pstats Documentation](https://docs.python.org/3/library/profile.html#module-pstats).

### Trace Plugin

The IOpipe agent comes bundled with a trace plugin that allows you to perform tracing.

Here's an example of how to use the trace plugin:

```python
from iopipe import IOpipe
from iopipe.contrib.trace import TracePlugin

iopipe = IOpipe(plugins=[TracePlugin()])

@iopipe
def handler(event, context):
    context.iopipe.mark.start('expensive operation')
    # do something here
    context.iopipe.mark.end('expensive operation')
```

Or you can use it as a context manager:

```python
from iopipe import IOpipe
from iopipe.contrib.trace import TracePlugin

iopipe = IOpipe(plugins=[TracePlugin()])

@iopipe
def handler(event, context):
    with context.iopipe.mark('expensive operation'):
        # do something here
```

Any block of code wrapped with `start` and `end` or using the context manager will be traced and the data collected will be available on your IOpipe dashboard.

By default, the trace plugin will auto-measure any trace you make. But you can disable this by setting `auto_measure` to `False`:

```python
from iopipe import IOpipe
from iopipe.contrib.trace import TracePlugin

iopipe = IOpipe(plugins=[TracePlugin(auto_measure=False)])

@iopipe
def handler(event, context):
    with context.iopipe.mark('expensive operation'):
        # do something here

    # Manually measure the trace
    context.iopipe.mark.measure('expensive operation')
```

### Creating Plugins

To create an IOpipe plugin you must implement the `iopipe.plugins.Plugin` interface.

Here is a minimal example:

```python
from iopipe.plugins import Plugin


class MyPlugin(Plugin):
    @property
    def name(self):
        return 'my-plugin'

    @property
    def version(self):
        return '0.1.0'

    @property
    def homepage(self):
        return 'https://github.com/iopipe/my-plugin/'

    def pre_setup(self, iopipe):
        pass

    def post_setup(self, iopipe):
        pass

    def pre_invoke(self, event, context):
        pass

    def post_invoke(self, event, context):
        pass

    def pre_report(self, report):
        pass

    def post_report(self):
        pass
```

As you can see, this plugin doesn't do much. If you want to see a functioning example of a plugin check out the trace plugin at `iopipe.contrib.trace.plugin.TracePlugin`.

#### Plugin Properties

A plugin has the following properties defined:

- `name`: The name of the plugin, must be a string.
- `version`: The version of the plugin, must be a string.
- `homepage`: The URL of the plugin's homepage, must be a string.

#### Plugin Methods

A plugin has the following methods defined:

- `pre_setup`: Is called once prior to the agent initialization. Is passed the `iopipe` instance.
- `post_setup`: Is called once after the agent is initialized, is passed the `iopipe` instance.
- `pre_invoke`: Is called prior to each invocation, is passed the `event` and `context` of the invocation.
- `post_invoke`: Is called after each invocation, is passed the `event` and `context` of the invocation.
- `pre_report`: Is called prior to each report being sent, is passed the `report` instance.
- `post_report`: Is called after each report is sent, is passed the `report` instance.

## Supported Python Versions

This package supports Python 2.7 and 3.6, the runtimes supported by AWS Lambda.

## Framework Integration

IOpipe integrates with popular serverless frameworks. See below for examples. If you don't see a framework you'd like to see supported, please create an issue.

### Chalice

Using IOpipe with the [Chalice](https://github.com/aws/chalice) framework is easy. Just wrap your `app` like so:

```python
from chalice import Chalice
from iopipe import IOpipe

iopipe = IOpipe()

app = Chalice(app_name='helloworld')

@app.route('/')
def index():
    return {'hello': 'world'}

# Do this after defining your routes
app = iopipe(app)
```

### Zappa

Using IOpipe with [Zappa](https://github.com/Miserlou/Zappa) is easy. In your project add the following:

```python
from iopipe import IOpipe
from zappa.handler import lambda_handler

iopipe = IOpipe()
lambda_handler = iopipe(lambda_handler)
```

Then in your `zappa_settings.json` file include the following:

```js
{
  "lambda_handler": "module.path.to.lambda_handler"
}
```

Where `module-path.to.lambda_handler` is the Python module path to the `lambda_handler` you created above. For example, if you put it in `myproject/__init__.py` the path would be `myproject.lambda_handler`.

## License

Apache 2.0
