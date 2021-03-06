---
title: Flask
sidebar_order: 7
---

<!-- WIZARD -->
## Installation

If you haven’t already, install raven with its explicit Flask dependencies:

```bash
pip install raven[flask]
```

## Setup

The first thing you’ll need to do is to initialize Raven under your application:

```python
from raven.contrib.flask import Sentry
sentry = Sentry(app, dsn='___DSN___')
```

If you don’t specify the `dsn` value, we will attempt to read it from your environment under the `SENTRY_DSN` key.
<!-- ENDWIZARD -->

## Extended Setup

You can optionally configure logging too:

```python
import logging
from raven.contrib.flask import Sentry
sentry = Sentry(app, logging=True, level=logging.ERROR, \
                logging_exclusions=("logger1", "logger2", ...))
```

Building applications on the fly? You can use Raven’s `init_app` hook:

```python
sentry = Sentry(dsn='http://public_key:secret_key@example.com/1')

def create_app():
    app = Flask(__name__)
    sentry.init_app(app)
    return app
```

You can pass parameters in the `init_app` hook:

```python
sentry = Sentry()

def create_app():
    app = Flask(__name__)
    sentry.init_app(app, dsn='___DSN___', logging=True,
                    level=logging.ERROR,
                    logging_exclusions=("logger1", "logger2", ...))
    return app
```

## Settings

Additional settings for the client can be configured using `SENTRY_CONFIG` in your application’s configuration:

```python
class MyConfig(object):
    SENTRY_CONFIG = {
        'dsn': '___DSN___',
        'include_paths': ['myproject'],
        'release': raven.fetch_git_sha(os.path.dirname(__file__)),
    }
```

If [Flask-Login](https://pypi.python.org/pypi/Flask-Login/) is used by your application (including [Flask-Security](https://pypi.python.org/pypi/Flask-Security/)), user information will be captured when an exception or message is captured. By default, only the `id` (current_user.get_id()), `is_authenticated`, and `is_anonymous` is captured for the user. If you would like additional attributes on the `current_user` to be captured, you can configure them using `SENTRY_USER_ATTRS`:

```python
class MyConfig(object):
    SENTRY_USER_ATTRS = ['username', 'first_name', 'last_name', 'email']
```

`email` will be captured as `sentry.interfaces.User.email`, and any additional attributes will be available under `sentry.interfaces.User.data`

You can specify the types of exceptions that should not be reported by Sentry client in your application by setting the `ignore_exceptions` configuration value:

```python
class MyExceptionType(Exception):
    def __init__(self, message):
        super(MyExceptionType, self).__init__(message)

app = Flask(__name__)
app.config['SENTRY_CONFIG'] = {
    'ignore_exceptions': [MyExceptionType],
}
```

## Usage

Once you’ve configured the Sentry application it will automatically capture uncaught exceptions within Flask. If you want to send additional events, a couple of shortcuts are provided on the Sentry Flask middleware object.

Capture an arbitrary exception by calling `captureException`:

```python
try:
    1 / 0
except ZeroDivisionError:
    sentry.captureException()
```

Log a generic message with `captureMessage`:

```python
sentry.captureMessage('hello, world!')
```

## Getting The Last Event ID

If possible, the last Sentry event ID is stored in the request context `g.sentry_event_id` variable. This allow to present the user an error ID if have done a custom error 500 page.

```html
{% raw %}<h2>Error 500</h2>
{% if g.sentry_event_id %}
<p>The error identifier is {{ g.sentry_event_id }}</p>
{% endif %}{% endraw %}
```

## User Feedback {#python-flask-user-feedback}

To enable user feedback for crash reports just make sure you have a custom _500_ error handler and render out a HTML snippet for bringing up the crash dialog:

```python
from flask import Flask, g, render_template
from raven.contrib.flask import Sentry

app = Flask(__name__)
sentry = Sentry(app, dsn='___DSN___')

@app.errorhandler(500)
def internal_server_error(error):
    return render_template('500.html',
        event_id=g.sentry_event_id,
        public_dsn=sentry.client.get_public_dsn('https')
    )
```

And in the error template (`500.html`) you can then do this:

```html
{% raw %}<!-- Sentry JS SDK 2.1.+ required -->
<script src="https://cdn.ravenjs.com/2.3.0/raven.min.js"></script>

{% if event_id %}
  <script>
  Raven.showReportDialog({
    eventId: '{{ event_id }}',
    dsn: '{{ public_dsn }}'
  });
  </script>
{% endif %}{% endraw %}
```

That’s it!

For more details on this feature, see the [_User Feedback guide_]({%- link _documentation/enriching-error-data/user-feedback.md -%}).

## Dealing With Proxies

When your Flask application is behind a proxy such as nginx, Sentry will use the remote address from the proxy, rather than from the actual requesting computer. By using `ProxyFix` from [werkzeug.contrib.fixers](http://werkzeug.pocoo.org/docs/0.14/contrib/fixers/#werkzeug.contrib.fixers.ProxyFix) the Flask `.wsgi_app` can be modified to send the actual `REMOTE_ADDR` along to Sentry.

```python
from werkzeug.contrib.fixers import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app)
```

This may also require [changes](http://flask.pocoo.org/docs/0.12/deploying/wsgi-standalone/#proxy-setups) to the proxy configuration to pass the right headers if it isn’t doing so already.

## Signals

Raven uses [blinker](https://github.com/jek/blinker) to emit a signal (called `logging_configured`) after logging has been configured for the client. You may [bind to that signal](https://pythonhosted.org/blinker/#subscribing-to-signals) in your application to do any additional configuration to the logging handler `SentryHandler`.

```python
from raven.contrib.flask import Sentry, logging_configured
from flask import Flask, g, render_template
from raven.contrib.flask import Sentry

app = Flask(__name__)
sentry = Sentry(app, dsn='___DSN___', logging=True)

@logging_configured.connect
def internal_server_error(sender, sentry_handler=None, **kwargs):
    # configure sentry_handler here
    sentry_handler.addFilter(some_filter)
```
