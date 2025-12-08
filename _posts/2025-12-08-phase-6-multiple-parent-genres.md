---
title: "Music Graph Project: Multiple Parent Genres"
date: 2025-12-08
categories:
  - projects
  - python
tags:
  - flask
  - music-graph
  - database
  - refactoring
---

***I was asked by my user community of one to create the ability to have multiple parent genres. This was a really tough refactor. At least for the way we tried it. I let claude write the files to my local clone of the git repo and it went badly. I ended up having to reset the local repo and make the changes line by line. All in all it could have gone worse. It was my idea to test it locally before pushing to GCP. Which turned out to be the best method. Below is the post written by claude. I am too tired to add screenshots***

Phase 6 implements multiple parent genre support, addressing a fundamental limitation in how genres relate to each other. Previously, each genre could only have one parent. Now genres can belong to multiple parent categories while maintaining a clean graph visualization.

## The Problem: Rigid Hierarchies

After Phase 5 deployment, Aidan started adding genres and immediately hit a limitation: Grindcore.

Grindcore evolved from both:
- **Metal** (particularly Death Metal and Thrash)
- **Hardcore Punk** (the aggressive DIY ethos)

The old system forced a choice: put Grindcore under Metal OR Hardcore, but not both. This misrepresented the actual musical relationships.

The same issue appeared with other crossover genres:
- Metalcore (Metal + Hardcore)
- Folk Metal (Metal + Folk)
- Jazz Fusion (Jazz + multiple influences)

Musical genres don't fit strict hierarchies. They're networks of influence and connection.

## The Solution: Primary + All Parents

We implemented the same pattern already used for Bands:

**For Bands (existing):**
- `primary_genre_id` → single genre (shows in graph)
- `genres` → many-to-many relationship (full list)

**For Genres (new):**
- `parent_id` → primary parent (shows in graph)
- `parent_genres` → many-to-many relationship (full list)

**Example:**
```python
{
    "id": "grindcore",
    "name": "Grindcore",
    "parent_id": "metal",  # Primary - visible in graph
    "parent_genres": ["metal", "hardcore"],  # Complete relationship data
    "type": "leaf"
}
```

The graph shows Grindcore connected only to Metal (clean visualization), but the database stores both parent relationships for accuracy and future features.

## Database Schema Changes

**Added association table:**
```python
genre_parents = db.Table('genre_parents',
    db.Column('genre_id', db.String(50), db.ForeignKey('genres.id'), primary_key=True),
    db.Column('parent_genre_id', db.String(50), db.ForeignKey('genres.id'), primary_key=True)
)
```

**Updated Genre model:**
```python
class Genre(db.Model):
    # ... existing columns ...
    parent_id = db.Column(...)  # Primary parent (unchanged)
    
    # Many-to-many for all parent genres
    parent_genres = db.relationship(
        'Genre',
        secondary=genre_parents,
        primaryjoin=(id == genre_parents.c.genre_id),
        secondaryjoin=(id == genre_parents.c.parent_genre_id),
        backref='child_genres'
    )
```

This creates a many-to-many relationship between genres and their parents, separate from the single `parent_id` foreign key.

## Form Updates

Both add and edit genre forms now have two parent selection fields:

**Primary Parent** (dropdown):
- Single selection
- Determines graph visualization
- Required for non-root genres

**All Parent Genres** (multi-select):
- Multiple selection (Ctrl/Cmd+click)
- Stores complete parent relationships
- Must include the primary parent

JavaScript auto-selects the primary parent in the multi-select when changed, preventing inconsistent state.

**Validation ensures:**
- Primary parent is always in the all-parents list
- Genres can't be their own parent (circular reference prevention)
- Invalid parent IDs are rejected

## Data Migration

Existing genres needed their `parent_id` migrated to the new `genre_parents` table without data loss.

**Migration script (`migrate_genre_parents.py`):**
1. Finds all genres with a `parent_id`
2. Adds that parent to `parent_genres` relationship
3. Preserves existing data
4. Verifies migration success

The migration ran successfully on both local development (SQLite) and production (PostgreSQL) databases, populating 15 existing parent relationships.

## Graph Visualization: No Changes

The graph visualization code didn't change at all. It still uses `parent_id` to draw connections, so:

**Before:** Grindcore → Metal (only option)

**After:** Grindcore → Metal (shown in graph) + Hardcore (stored in database)

Clean visualization maintained while adding data accuracy.

## Development Workflow Evolution

This phase marked a shift in development approach:

**Previous phases:** Full code generation, then manual verification

**This phase:** Methodical step-by-step changes
1. Update models.py → test
2. Update add_genre() → test
3. Update add_genre.html → test
4. Update edit_genre() → test
5. Update edit_genre.html → test
6. Create migration script → test

Each change was small, testable, and verified before moving forward. When a syntax error appeared, we rolled back and restarted cleanly using feature branches.

This slower approach proved essential for refactoring existing, working code. For new features, full-file generation might still be faster.

## Local Testing First

All changes were developed and tested locally before deployment:

**Local environment:**
- SQLite database (lightweight)
- No production impact
- Fast iteration

**Feature branch workflow:**
```bash
git checkout -b feature/multiple-parent-genres
# Make changes
# Test thoroughly
git commit -m "Add multiple parent genres support"
git checkout main
git merge feature/multiple-parent-genres
```

Only after local verification did we deploy to GCP production.

## Production Deployment

**Deployment steps:**
1. SSH to GCP VM
2. Pull latest code from GitHub
3. Create new database table in container:
   ```bash
   docker-compose exec web python -c "from app import app; from models import db; app.app_context().push(); db.create_all()"
   ```
4. Run migration inside container:
   ```bash
   docker-compose exec web python migrate_genre_parents.py
   ```
5. Rebuild and restart containers
6. Verify production functionality

The PostgreSQL database gained the `genre_parents` table, and all existing parent relationships migrated successfully.

## What This Enables

**Immediate benefits:**
- Accurate representation of crossover genres
- Foundation for future detail views
- Better metadata for exploration features

**Future possibilities (from planning):**
- Click a genre → see all parent connections
- Genre detail pages showing influences
- "Related genres" features
- Search/filter by genre characteristics
- Network visualization of genre evolution

The data structure now supports these features without further schema changes.

## Lessons Learned

**Refactoring requires patience:** The methodical, step-by-step approach was frustrating compared to earlier rapid development, but essential for maintaining code quality.

**Feature branches are worth it:** Rolling back and restarting cleanly saved time compared to trying to fix broken state.

**Test locally first:** Catching issues in SQLite before PostgreSQL deployment prevented production downtime.

**UI consistency matters:** Mirroring the Band pattern (primary + all) made the system more intuitive. Users already understand how Bands work with multiple genres.

**Migration scripts are code:** The migration script needed the same care as application code - error handling, verification, user confirmation.

## Current State

The application now supports complex genre relationships while maintaining visual simplicity. Aidan can accurately categorize crossover genres, and the foundation exists for richer exploration features.

**Live at:** https://music-graph.billgrant.io

Try adding a genre with multiple parents - the form makes it clear which is primary (graph visualization) versus all parents (complete data).

## What's Next: Phase 7

Original plan was CI/CD and DevOps (GitHub Actions, automated deployments, monitoring). That's still important but might be delayed.

Aidan's use of the application is revealing other needs:
- Password management (Issue #1)
- Production WSGI server (Issue #2)
- Detail views for genres and bands

The roadmap adapts based on actual usage. That's the benefit of having a real user testing the application.

## Code

This release is tagged as [v0.5.0-alpha](https://github.com/billgrant/music-graph/releases/tag/v0.5.0-alpha).

Changes visible in [GitHub Issue #3](https://github.com/billgrant/music-graph/issues/3).

---

*This is part of the [Music Genre Graph project series](https://billgrant.io/tags/#music-graph). See the [project introduction](https://billgrant.io/2025/11/24/music-graph-intro/) for the full roadmap.*
