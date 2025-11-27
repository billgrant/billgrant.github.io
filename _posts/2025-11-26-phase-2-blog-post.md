---
title: "Music Graph Project: Database Integration"
date: 2025-11-26
categories:
  - projects
  - python
  - database
tags:
  - flask
  - music-graph
  - sqlalchemy
  - sqlite
---

***Claude and I were able to sucessfully move to sqlite from a file with a `dict` in it. I hit my session limit with Claude ($20 plan) tonight. Mostly because I was asking multiple questions for each step. All in all this was a pretty easy move but, I don't like that we are converting the sql data to a dict instead of accessing the db directly. This was done so that we didn't break the current javascript. We are going to fix that later. Below is a blog post written by Claude. The only edit from me is adding the screenshot.***

Phase 2 is complete - the application now uses a proper database instead of hardcoded dictionaries. Everything still works exactly the same from a user perspective, but the foundation is much stronger.

## Why Move to a Database?

Phase 1 proved the concept with Python dictionaries. But dictionaries have limitations:
- Data disappears when the app restarts
- Adding content requires editing code
- No validation or constraints
- Difficult to query or filter
- Can't support multiple users editing simultaneously

A database solves all of these problems.

## The Migration

**From this (dictionaries):**
```python
genres = {
    "death-metal": {
        "name": "Death Metal",
        "connections": ["metal"]
    }
}
```

**To this (SQLAlchemy models):**
```python
class Genre(db.Model):
    id = db.Column(db.String(50), primary_key=True)
    name = db.Column(db.String(100), nullable=False)
    parent_id = db.Column(db.ForeignKey('genres.id'))
    type = db.Column(db.String(20), nullable=False)
```

## Implementing Proper Hierarchy

Phase 1 used a `connections` field that treated all relationships as equal. But our domain model is actually hierarchical:

- Rock (root genre)
  - Metal (intermediate - organizes sub-genres)
    - Death Metal (leaf - bands belong here)
    - Groove Metal (leaf)
    - Thrash Metal (leaf)

The database schema now reflects this properly:

**Genre Types:**
- `root` - Top-level genres like Rock, Electronic
- `intermediate` - Parent genres like Metal (organize but don't contain bands)
- `leaf` - Actual genres bands belong to (Death Metal, Groove Metal)

**Parent-Child Relationships:**
```python
parent = db.relationship('Genre', remote_side=[id], backref='children')
```

This enables queries like "get all children of Metal" or "get the full path from Death Metal to Rock."

## Band Relationships

Bands maintain both primary genre (for graph visualization) and full genre list (for metadata):

```python
class Band(db.Model):
    primary_genre_id = db.Column(db.ForeignKey('genres.id'))
    genres = db.relationship('Genre', secondary='band_genres')
```

The `band_genres` junction table handles the many-to-many relationship automatically. Pantera can be tagged with both Groove Metal and Thrash Metal, but only connects to Groove Metal in the graph.

## SQLAlchemy Basics

SQLAlchemy is an ORM (Object-Relational Mapper) - it lets you work with database tables as Python objects.

**Instead of SQL:**
```sql
SELECT * FROM genres WHERE parent_id = 'metal';
```

**You write Python:**
```python
Genre.query.filter_by(parent_id='metal').all()
```

**Understanding the layers:**
- SQLite = the actual database engine (built into Python)
- SQLAlchemy = Python library that talks to SQLite (and other databases)
- Flask-SQLAlchemy = Integration between Flask and SQLAlchemy

SQLAlchemy uses "dialects" to speak to different databases. Change the database URL from `sqlite:///` to `postgresql://` and most of your code stays the same. This makes migration to PostgreSQL easy when needed.

## Initializing the Database

Created `init_db.py` to set up and populate the database:

```python
def init_database():
    with app.app_context():
        db.drop_all()
        db.create_all()
        
        # Create genres with hierarchy
        metal = Genre(id='metal', name='Metal', 
                     parent_id='rock', type='intermediate')
        
        death_metal = Genre(id='death-metal', name='Death Metal',
                          parent_id='metal', type='leaf')
        
        # Create bands with relationships
        pantera = Band(id='pantera', name='Pantera',
                      primary_genre_id='groove-metal')
        pantera.genres = [groove_metal, thrash_metal]
        
        db.session.add_all([metal, death_metal, pantera])
        db.session.commit()
```

Run `python init_db.py` and you get a fully populated `music_graph.db` file.

## Preserving Functionality

The graph still works exactly the same. Flask routes now query the database:

```python
@app.route('/')
def index():
    genres = Genre.query.all()
    bands = Band.query.all()
    # Convert to dict format for template compatibility
    return render_template('index.html', genres=genres_dict, bands=bands_dict)
```

We're temporarily converting SQLAlchemy objects back to dictionaries so the existing templates don't break. This is intentional - change one thing at a time, verify it works, then optimize later.

## What This Enables

With a proper database, we can now:
- Add CRUD operations (create/edit/delete through web forms)
- Implement user authentication
- Support multiple users editing simultaneously
- Add validation (prevent invalid data)
- Query and filter efficiently
- Eventually migrate to PostgreSQL if needed

## Current State

The application now has:
- Persistent data storage (survives restarts)
- Proper genre hierarchy (root/intermediate/leaf)
- Band relationships (primary + full genre list)
- Foundation for user contributions
- Same graph visualization and interaction as Phase 1

Everything works, but now it's backed by a real database.

## Screenshot

![movedtodb](/assets/images/music-graph-screenshot-2025-11-26-3.png)

## What's Next: Phase 3

CRUD operations - adding web forms so genres and bands can be added/edited without touching code or database directly. This makes the application actually usable by non-developers.

Eventually, Aidan (my son who designed the expand/collapse interaction) will be able to log in and add bands himself.

## Code

This release is tagged as [v0.1.0-alpha](https://github.com/billgrant/music-graph/releases/tag/v0.1.0-alpha).

---

*This is part of the [Music Genre Graph project series](https://billgrant.io/tags/#music-graph). See the [project introduction](https://billgrant.io/2025/11/24/music-graph-intro/) for the full roadmap.*
