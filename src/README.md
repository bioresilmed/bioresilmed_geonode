# GeoNode project – frontend customization (templates, styles, i18n)

This repository is a GeoNode project (legacy name: `bioresilmed_geonode`) prepared to customize the UI **outside** the Docker image. You can add or override **templates**, **styles**, and **translations** directly under `src/` and see the changes live.

## Quick start

```bash
# build and start the stack
docker compose up -d --build

# follow logs during first run
docker compose logs -f django

# open a shell in the django container
docker compose exec django bash

# restart only django (picks up settings/template changes)
docker compose restart django
```

## Project layout

```
src/
└─ bioresilmed_geonode/
   ├─ bioresilmed_geonode/
   │  ├─ settings.py
   │  ├─ static/
   │  │  ├─ css/
   │  │  │  └─ site_base.css
   │  │  └─ img/           # logo.svg, hero.jpg, favicons, etc.
   │  ├─ templates/
   │  │  ├─ index.html
   │  │  └─ geonode-mapstore-client/
   │  │     └─ snippets/   # header.html, footer.html, hero.html, custom_theme.html, …
   │  └─ locale/
   │     ├─ es/LC_MESSAGES/django.po, django.mo
   │     └─ ca/LC_MESSAGES/django.po, django.mo  # optional
   └─ …
```

> The stock templates from the container have been copied here as a base. Keep only those you actually customize.

## How template overriding works

Django is configured to search your project templates first:

```python
# bioresilmed_geonode/settings.py
LOCAL_ROOT = os.path.abspath(os.path.dirname(__file__))
TEMPLATES[0]["DIRS"].insert(0, os.path.join(LOCAL_ROOT, "templates"))
```

If your path matches a core template (for example `index.html` or `geonode-mapstore-client/snippets/header.html`), your file wins. No image rebuild is needed in this setup; a container restart is enough if the change is not picked up automatically.

### Useful inspection commands

```bash
# which template is actually in use
docker compose exec django bash -lc \
"python -c \"from django.template.loader import get_template; print(get_template('index.html').origin.name)\""

# where a static file is served from
docker compose exec django bash -lc "python manage.py findstatic css/site_base.css"
```

## Styles (css, logo, favicons)

Place custom css in `static/css/site_base.css` and small, site-wide tweaks in `templates/geonode-mapstore-client/snippets/custom_theme.html` (loaded on every page; great for css variables).

Put your logo and icons in `static/img/` and make sure `snippets/head.html` references them:

```django
{% block favicon %}
<link rel="icon" type="image/png" sizes="32x32" href="{% static 'img/favicon.ico' %}">
<link rel="apple-touch-icon" sizes="180x180" href="{% static 'img/apple-touch-icon.png' %}">
<link rel="icon" type="image/png" sizes="192x192" href="{% static 'img/android-chrome-192x192.png' %}">
<link rel="icon" type="image/png" sizes="512x512" href="{% static 'img/android-chrome-512x512.png' %}">
{% endblock %}
```

When you add **new** files under `static/`, collect statics:

```bash
docker compose exec django bash -lc "python manage.py collectstatic --noinput"
```

Editing existing files with the same path usually only needs a hard reload in the browser (Ctrl/Cmd+Shift+R).

## Homepage intro block

The homepage adds a short introduction above the resource grids. Edit `templates/index.html`:

```django
<div class="gn-container">
  <div class="gn-content">
    <section class="lf-intro" aria-label="{% trans 'site introduction' %}">
      <h2 class="lf-intro__title">{% trans "Sharing experimental field data" %}</h2>
      <p class="lf-intro__lead">
        {% blocktrans trimmed %}
        A community repository for georeferenced experimental data — plots, agroecological crops,
        new treatments and management practices. We publish maps, datasets and documents with clear
        metadata so research is easier to find, reuse and compare. Contribute your trials and help
        build an open evidence base for sustainable agriculture.
        {% endblocktrans %}
        <a class="lf-intro__link" href="/about/">{% trans "learn more" %}</a>
      </p>
    </section>
  </div>
</div>
```

The minimal css for this block lives in the page’s `{% block extra_style %}` or your `custom_theme.html`.

## Django translations (server-side templates)

### Create or update messages

```bash
# create/update .po files (spanish + catalan)
docker compose exec django bash -lc "
cd /usr/src/bioresilmed_geonode && \
django-admin makemessages --no-location \
  -l es -l ca -d django -e 'html,txt,py' \
  -i 'docs' -i 'static' -i 'node_modules'
"
```

Edit the `.po` files under `locale/<lang>/LC_MESSAGES/django.po`, then compile:

```bash
docker compose exec django bash -lc "
cd /usr/src/bioresilmed_geonode && django-admin compilemessages
"
```

To avoid root-owned files on the host, run with your uid/gid:

```bash
docker compose exec --user $(id -u):$(id -g) django bash -lc "
cd /usr/src/bioresilmed_geonode && \
django-admin makemessages --no-location -l es -l ca -d django -e 'html,txt,py' \
  -i docs -i static -i node_modules && \
django-admin compilemessages
"
```

### Enable languages

Languages are set in `.env`, for example:

```
LANGUAGE_CODE=es-ES
LANGUAGES=(('en','English'),('es','Español'),('ca','Català'))
```

Restart django after changing languages:

```bash
docker compose restart django
```

## MapStore (spa) translations

MapStore UI strings (for example “View”, “Order by”, etc.) are translated via JSON bundles. You can override them without rebuilding MapStore.

Create project overrides:

```
static/mapstore/project-translations/
├─ data.en-US.json
├─ data.es-ES.json
└─ data.ca-ES.json    # optional
```

Example:

```json
{
  "locale": "es-ES",
  "messages": {
    "gnhome": {
      "view": "Abrir"
    }
  }
}
```

Tell GeoNode where to look in `settings.py`:

```python
MAPSTORE_TRANSLATIONS_PATH = [
    '/static/mapstore/ms-translations',
    '/static/mapstore/gn-translations',
    '/static/mapstore/project-translations',  # your overrides
]
```

If you added new files, collect statics:

```bash
docker compose exec django bash -lc "python manage.py collectstatic --noinput"
```

## Working inside Docker

```bash
# open a shell in django and explore
docker compose exec django bash

# run migrations (if you enable extra contrib apps)
docker compose exec django bash -lc "python manage.py migrate"

# restart services when needed
docker compose restart django
docker compose restart geonode
docker compose restart geoserver
```

## Troubleshooting

If a template change does not show, check which file is actually loaded:

```bash
docker compose exec django bash -lc \
"python -c \"from django.template.loader import get_template; print(get_template('index.html').origin.name)\""
```

When adding new static assets, run:

```bash
docker compose exec django bash -lc "python manage.py collectstatic --noinput"
```

After enabling a new language, ensure `.mo` files exist and restart django:

```bash
docker compose exec django bash -lc "cd /usr/src/bioresilmed_geonode && django-admin compilemessages"
docker compose restart django
```

Use a hard reload in the browser to bypass cached js/css (Ctrl/Cmd+Shift+R).
