---
title: "Music Graph Project: Getting Flask Running"
date: 2025-11-24
categories:
  - projects
  - python
tags:
  - flask
  - music-graph
  - web-development
---

***I built this shell of my application doing pair programming with claude.ai. Claude wrote the majority of the code, but I asked for detailed explanations on anything I didn't understand. Below is the post claude wrote recapping our session. I edited it a bit and added the screenshot***

First real progress on the music graph project - got a basic Flask application running that displays genre and band data.

## What I Built

A simple Flask app that renders the genre/band data structures I defined in the project intro. Nothing fancy yet - just cards showing genres, their connections, and bands with their associated genres.

**Project structure:**
```
music-graph/
├── app.py              # Flask application
├── data.py             # Genres and bands dictionaries
├── templates/
│   └── index.html      # Main template
└── static/
    └── style.css       # Dark theme styling
```

## Key Concepts

**Routes and rendering:**
```python
@app.route('/')
def index():
    return render_template('index.html', genres=genres, bands=bands)
```

The decorator tells Flask "when someone visits `/`, run this function." The function passes data to the template, which Jinja2 processes into HTML.

**Template loops:**
```html
{% raw %}
{% for genre_id, genre_data in genres.items() %}
    <h3>{{ genre_data.name }}</h3>
{% endfor %}
{% endraw %}
```

Jinja2 iterates through the data and generates HTML for each item. Change the data in `data.py`, refresh the page, and it updates automatically.

## Current State

Visit `localhost:5000` and you see:
- All genres displayed as cards
- Each genre shows its connections to other genres
- All bands displayed with their associated genres
- Dark theme matching the blog

It's basic, but it works. The data structure flows from Python dictionaries → Flask → Jinja2 templates → rendered HTML.

## Screenshot

![musicgraphpoc](assets/images/music-graph-screenshot-2025-11-24.png)

## What I Learned

- Flask's `@app.route()` decorator maps URLs to functions
- `render_template()` passes data to Jinja2 templates
- Templates use `{% raw %}{{ }}{% endraw %}` for variables and `{% raw %}{% %}{% endraw %}` for control flow
- The development flow: change data → save → refresh → see results

## Next Steps

Phase 1 isn't done yet. Next session I'll research graph visualization libraries and figure out how to turn these cards into an actual connected graph view. That's the interesting part - making it visual and interactive.

For now, the foundation is solid: Flask is running, data is displaying, and I understand how the pieces connect.

---

*This is part of the [Music Genre Graph project series]([music-graph tag]({{ site.url }}/tags/#music-graph)). See the [project introduction]({% post_url 2025-11-24-music-graph-intro %}) for the full roadmap.*
