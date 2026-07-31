---
name: plugin_page_pattern
description: How to build an IvoryOS plugin page — Flask Blueprint, base.html integration, SocketIO, and hardware access
---

# IvoryOS Plugin Page Pattern

This skill covers everything you need to build a working IvoryOS plugin page from the
template in this repo. Read it before writing any route, template, or SocketIO handler.

---

## 1. What a Plugin Is

An IvoryOS plugin is a standard **Flask Blueprint** packaged as a Python module and injected
into the IvoryOS server at startup:

```python
# In the user's lab script
import ivoryos
from ivoryos_plugin.plugin import plugin

ivoryos.run(module=__name__, blueprint_plugins=plugin)
```

IvoryOS mounts it at `/ivoryos/<blueprint_name>/` and adds it to the navigation bar automatically.

---

## 2. File Structure

```
ivoryos_plugin/
  __init__.py
  plugin.py          ← Blueprint definition + routes + optional SocketIO
  templates/
    example.html     ← Your Jinja2 template (MUST handle base.html — see section 4)
```

---

## 3. Defining the Blueprint (`plugin.py`)

```python
import os
from flask import Blueprint, render_template, current_app

# Blueprint name must be unique — avoid IvoryOS reserved names:
# "auth", "control", "data", "design", "execute", "library", "main"
plugin = Blueprint(
    "my_plugin",          # ← choose a unique name
    __name__,
    template_folder=os.path.join(os.path.dirname(__file__), "templates")
)

# Optional: control where the nav link appears
# plugin.plugin_type = "tab"        # default — appears in top nav
# plugin.plugin_type = "left_panel" # appears in sidebar
```

The **`main()` route** (at `/`) is the entry point shown in the IvoryOS nav:

```python
@plugin.route('/', strict_slashes=False)
def main():
    base_exists = "base.html" in current_app.jinja_loader.list_templates()
    return render_template('example.html', base_exists=base_exists)
```

The `base_exists` flag lets the template gracefully degrade when run standalone (without IvoryOS).

---

## 4. The `base.html` Trick — Most Common Agent Mistake

IvoryOS's nav bar, sidebar, CSS, and JS are all provided by `base.html` from the main app.
Your template **must** conditionally extend it:

```html
{% if base_exists %}
    {% extends 'base.html' %}
    {% block title %}My Plugin{% endblock %}
{% else %}
    <!-- Standalone fallback when running without IvoryOS -->
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0-alpha2/dist/css/bootstrap.min.css"
              integrity="sha384-aFq/bzH65dt+w6FI2ooMVUpc+21e0SRygnTpmBvdBgSdnuTN7QbdgL+OapgHtvPp" crossorigin="anonymous">
    </head>
{% endif %}

{% block body %}
<div class="container mt-4">
    <!-- Your plugin content here -->
</div>
{% endblock %}
```

**Why this pattern?** The `base.html` loader is only available when the plugin runs *inside*
IvoryOS (`current_app.jinja_loader` merges all blueprint loaders). When the plugin is run
as a standalone Flask app during development, `base.html` does not exist and a plain HTML
fallback is used instead.

> **Never** hardcode `{% extends 'base.html' %}` without the `{% if base_exists %}` guard —
> the plugin will crash when run standalone.

---

## 5. Adding SocketIO (Optional)

```python
# plugin.py
_socketio = None

def init_socketio(sio):
    """Called automatically by IvoryOS when the plugin is loaded."""
    global _socketio
    _socketio = sio

    @sio.on('my_event')
    def handle_my_event(data):
        sio.emit('my_response', {'result': data})
```

IvoryOS detects `init_socketio` automatically via `hasattr(blueprint, 'init_socketio')` in
`ivoryos.server.load_plugins()` — no extra registration needed.

Emit from a route:

```python
@plugin.route('/trigger')
def trigger():
    if _socketio:
        _socketio.emit('my_event', {'msg': 'hello'})
    return '', 204
```

---

## 6. Accessing IvoryOS Hardware (Deck)

```python
from ivoryos.runtime.runner_runtime import ensure_deck

@plugin.route('/read')
def read_sensor():
    deck = ensure_deck()                        # raises RuntimeError if no deck configured
    value = deck.my_instrument.read_value()    # call any public instrument method
    return jsonify({'value': value})
```

`ensure_deck()` always fetches the live `global_state.deck` — never cache the result in a
module-level variable.

---

## 7. Running Standalone (Development)

```python
# bottom of plugin.py
if __name__ == '__main__':
    from flask import Flask
    app = Flask(__name__)
    app.register_blueprint(plugin)
    app.run(debug=True)
```

In standalone mode, `base_exists` will be `False` and the fallback HTML head is used.
The plugin UI still works; it just lacks the IvoryOS nav bar.

---

## 8. Registering with IvoryOS

```python
# user's lab entry-point script
import ivoryos
from ivoryos_plugin.plugin import plugin as my_plugin

ivoryos.run(
    module=__name__,
    blueprint_plugins=my_plugin,          # single plugin
    # blueprint_plugins=[my_plugin, other] # or a list
)
```

The plugin is mounted at `/ivoryos/<blueprint_name>/` and appears in the nav automatically.

---

## 9. Reserved Blueprint Names

These names are already used by IvoryOS core — pick something different:

`auth`, `control`, `data`, `design`, `execute`, `library`, `main`

---

## 10. Checklist Before Finishing

- [ ] Blueprint name is unique (not in the reserved list above)
- [ ] Template uses `{% if base_exists %}{% extends 'base.html' %}{% endif %}` guard
- [ ] `main()` route exists at `/` as the nav entry point
- [ ] Hardware access uses `ensure_deck()`, not a cached deck reference
- [ ] `init_socketio(sio)` defined if SocketIO events are needed
- [ ] Standalone `if __name__ == '__main__'` block works without IvoryOS
