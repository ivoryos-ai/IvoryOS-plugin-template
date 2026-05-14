# IvoryOS plugin template

This repository is a starter template for building plugin pages for [IvoryOS](https://gitlab.com/heingroup/ivoryos).

Plugins are Flask `Blueprint` objects that IvoryOS loads as additional UI pages. Use a plugin when you want to add a monitoring page, visualization, dashboard, custom control panel, analysis view, or any other page that should live alongside the core IvoryOS workflow UI.

## Features

- Adds standalone pages to IvoryOS through Flask blueprints.
- Can reuse the IvoryOS base template when running inside IvoryOS.
- Can run as a standalone Flask app during development.
- Can optionally attach to the shared Flask-SocketIO server.
- Can access the active IvoryOS deck and runtime state when loaded inside IvoryOS.

## Repository structure

```text
ivoryos_plugin/
|-- templates/
|   `-- example.html
|-- __init__.py
`-- plugin.py

README.md
pyproject.toml
requirements.txt
setup.py
```

## Quick start

Clone the template and install IvoryOS:

```bash
git clone https://gitlab.com/heingroup/ivoryos-suite/ivoryos-plugin-template
cd ivoryos-plugin-template
pip install ivoryos
```

For local plugin development, install the template in editable mode:

```bash
pip install -e .
```

## Register the plugin with IvoryOS

Import the plugin blueprint and pass it to `ivoryos.run(...)`:

```python
from ivoryos_plugin.plugin import plugin

import ivoryos

ivoryos.run(__name__, blueprint_plugins=plugin)
```

Multiple plugins can be registered as a list:

```python
ivoryos.run(__name__, blueprint_plugins=[plugin, another_plugin])
```

IvoryOS registers each plugin under:

```text
/ivoryos/<blueprint_name>
```

For the default `Blueprint("plugin", ...)`, the plugin page is available at:

```text
/ivoryos/plugin
```

## Minimal `plugin.py`

Each plugin must expose a Flask `Blueprint`. The blueprint name must be unique and should not collide with IvoryOS built-in blueprints such as `auth`, `control`, `data`, `design`, `execute`, `library`, or `main`.

```python
import os

from flask import Blueprint, current_app, render_template

plugin = Blueprint(
    "plugin",
    __name__,
    template_folder=os.path.join(os.path.dirname(__file__), "templates"),
)


@plugin.route("/")
def main():
    base_exists = "base.html" in current_app.jinja_loader.list_templates()
    return render_template("example.html", base_exists=base_exists)
```

## Template pattern

The example template can extend the IvoryOS base template when the plugin is running inside IvoryOS, and fall back to a complete HTML page when running standalone.

```jinja
{% if base_exists %}
    {% extends "base.html" %}
    {% block title %}Plugin{% endblock %}
{% else %}
    <!doctype html>
    <html lang="en">
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <title>Plugin</title>
    </head>
    <body>
{% endif %}

{% block body %}
<div class="container mt-4">
    <h1>IvoryOS plugin page</h1>
</div>
{% endblock %}

{% if not base_exists %}
    </body>
    </html>
{% endif %}
```

## Access the active IvoryOS deck

When the plugin is loaded inside IvoryOS, the active deck is available from the shared runtime state.

```python
from ivoryos import global_state


@plugin.route("/status")
def status():
    deck = global_state.deck
    if deck is None:
        return {"online": False, "message": "No active deck"}

    return {
        "online": True,
        "deck": getattr(deck, "__name__", "deck"),
    }
```

Access hardware defensively. A plugin can be opened in offline mode or before a specific instrument has been initialized.

```python
deck = global_state.deck
balance = getattr(deck, "balance", None) if deck else None
```

## Optional Socket.IO integration

If your plugin needs websocket events, define an `init_socketio(...)` function and attach it to the blueprint object. IvoryOS calls this function when loading the plugin.

```python
socketio = None


def init_socketio(sio):
    global socketio
    socketio = sio

    @socketio.on("plugin_ping")
    def handle_plugin_ping(data):
        socketio.emit("plugin_pong", data)


plugin.init_socketio = init_socketio
```

## Run standalone during development

The plugin can run outside IvoryOS for layout and route development.

```python
from flask import Flask

if __name__ == "__main__":
    app = Flask(__name__)
    app.register_blueprint(plugin)
    app.run(debug=True)
```

For standalone Socket.IO development:

```python
from flask import Flask
from flask_socketio import SocketIO

if __name__ == "__main__":
    app = Flask(__name__)
    app.register_blueprint(plugin)

    socketio = SocketIO(app)
    init_socketio(socketio)
    socketio.run(app, debug=True, allow_unsafe_werkzeug=True)
```

## Development prompts

This template works well with coding assistants. Example prompts:

- "Build an IvoryOS plugin page that shows a live webcam stream."
- "Convert this existing HTML page into an IvoryOS plugin template."
- "Add a Socket.IO event to stream instrument status into the plugin page."
- "Add a route that reads the current IvoryOS deck and shows the state of `deck.balance`."

## Notes

- Keep the blueprint name unique.
- Keep plugin routes relative to the plugin blueprint.
- Avoid assuming hardware is connected; check `global_state.deck` and instrument attributes first.
- Keep reusable plugin logic inside the plugin package instead of editing IvoryOS core files.
