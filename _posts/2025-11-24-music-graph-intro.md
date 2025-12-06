---
title: "Building a Music Genre Graph: Project Introduction"
date: 2025-11-24
categories:
  - projects
  - python
tags:
  - flask
  - web-development
  - learning-in-public
  - music-graph
---

***I am going to work on a project with Claude.ai. It is personal music website I have been thinking about for a while. The details are in the post below. I told claude my idea and it made the post below. There were few edit requests and additions from me, however, the post has not "human" edited at all. I plan to do most of the coding myself with Claude assisting*** 

I'm starting a new project to build an interactive web application that visualizes music genres and bands as a connected graph. Think circles and lines - "Rock" connects to "Metal," which connects to "Death Metal" and "Groove Metal," and those connect to specific bands like Pantera.

This isn't about building something perfect. It's about learning modern web development, refreshing my Python skills, and documenting the entire process - successes, failures, and everything in between.

## The Vision

I want to create a visual map of how music genres relate to each other and which bands belong to which genres. The end goal is an interactive graph where you can:

- See genre relationships at a glance
- Click on a genre to see bands within it
- Explore connections between related genres
- Eventually, allow users to contribute and expand the data

Think of it as a knowledge graph for music, but starting simple and building complexity over time.

## Why This Project?

Several reasons:

**1. It's visual and interesting**
Music is something I care about, and a graph visualization is more engaging than typical CRUD apps.

**2. It touches multiple technologies**
This project will force me to work with frontend, backend, databases, authentication, containerization, and deployment - basically a full-stack learning experience.

**3. It's modular**
I can build it in small, manageable pieces. Each piece is a learning milestone and a blog post.

**4. It's a perfect AI collaboration project**
I'll be using Claude and other AI tools throughout development. This fits perfectly with this blog's theme of learning with AI assistance.

## The Technology Stack (Planned)

Starting simple and adding complexity:

**Backend:**
- Python with Flask (familiar enough to get started, but rusty)
- SQLite initially, maybe PostgreSQL later
- REST API for data access

**Frontend:**
- Basic HTML/CSS to start
- JavaScript for interactivity
- D3.js or similar for graph visualization (this will be a learning curve)

**Infrastructure:**
- Docker for containerization
- Initial deployment to homelab (learning experience)
- Eventually migrate to a CSP (AWS/GCP/Azure) if it becomes something worth maintaining long-term
- Monitoring with Grafana and/or ELK stack

**Data:**
- Start with a hardcoded dictionary
- Move to a proper database
- Eventually add user authentication for community contributions

## The Roadmap

I'm breaking this into phases. Each phase will have one or more blog posts documenting the work:

### Phase 1: Basic Flask Application
- Set up Flask project structure
- Create simple routes
- Render hardcoded genre/band data from a Python dictionary
- Basic HTML templates

**Learning focus:** Flask fundamentals, Python project structure, templating

### Phase 2: Database Integration
- Design a schema for genres, bands, and relationships
- Set up SQLite
- Migrate from dictionary to database queries
- Basic CRUD operations

**Learning focus:** Database design, SQL, ORM basics

### Phase 3: Graph Visualization
- Research visualization libraries
- Implement interactive graph display
- Connect frontend visualization to backend data
- Make it actually look like a graph, not just lists

**Learning focus:** JavaScript, data visualization, frontend/backend integration

### Phase 4: User Authentication
- Add user accounts
- Implement login/logout
- Allow authenticated users to suggest bands/genres
- Basic moderation or approval workflow

**Learning focus:** Authentication patterns, session management, security basics

### Phase 5: Containerization
- Create Dockerfile
- Set up docker-compose for local development
- Document the containerized setup

**Learning focus:** Docker, container orchestration

### Phase 6: API Layer
- Build REST API endpoints
- Document API with proper specs
- Consider versioning strategy

**Learning focus:** API design, REST principles, documentation

### Phase 7: Homelab Deployment
- Deploy to my homelab environment
- Configure networking and access
- Set up reverse proxy (Nginx or Traefik)
- SSL certificates and security hardening

**Learning focus:** Self-hosting, homelab infrastructure, networking

### Phase 8: Monitoring and Observability
- Set up Grafana dashboards for application metrics
- Configure logging (ELK stack or similar)
- Application performance monitoring
- Alerting for issues

**Learning focus:** Observability, monitoring tools, log aggregation

### Phase 9: CI/CD Pipeline
- Set up GitHub Actions
- Automated testing
- Automated deployment to homelab
- Rollback strategies

**Learning focus:** DevOps automation, testing, deployment pipelines

### Future Possibilities
- Cloud migration when/if needed (moving from homelab to CSP)
- Recommendation engine ("If you like X, try Y")
- Spotify integration for audio previews
- Advanced graph algorithms (shortest path between genres?)
- Mobile-responsive design improvements
- Community features (voting, discussions)

## My Current Skill Level

Being honest about where I'm starting:

**Strong:**
- Infrastructure and cloud concepts (this is my day job)
- Command line comfort
- General programming logic

**Rusty:**
- Python - I've used it before but it's been a while
- Web frameworks - Flask specifically
- Frontend development - very basic HTML/CSS, minimal JavaScript

**Need to Learn:**
- Graph visualization libraries
- Modern authentication patterns for web apps
- Docker in practice (understand concepts, less hands-on experience)
- CI/CD pipelines from scratch

## The AI Angle

Throughout this project, I'll be using AI tools (primarily Claude) as a development partner. This means:

- Planning architecture and discussing trade-offs
- Debugging issues and understanding error messages
- Learning new libraries and frameworks faster
- Getting code examples and explanations
- Reviewing my code for improvements

Each blog post will document how AI assistance helped (or didn't help) with that particular phase. The goal is to learn how to effectively collaborate with AI tools while building real technical skills.

## What to Expect

This project will take weeks or months. Posts will document:

- **What I built** - code snippets, architecture decisions, progress updates
- **What I learned** - new concepts, "aha" moments, resources that helped
- **What went wrong** - bugs, wrong turns, things I had to redo
- **How AI helped** - specific ways Claude or other tools assisted

Some posts will be short updates. Others will be deeper dives into specific problems. The point isn't polished tutorials - it's real documentation of a learning process.

## Starting Tomorrow

The next post will cover setting up the basic Flask application and rendering a simple hardcoded music graph as a proof of concept. Nothing fancy - just getting something running locally.

If you're interested in following along, the code will be on [GitHub](https://github.com/billgrant/music-graph) 

Let's see where this goes.

## Progress Updates

**Phase 4 Complete! ✅**

The application now has user authentication and role-based access control. Only admins can modify data, making it secure for multi-user deployment.

**Phase 1 Posts:**
- [Getting Flask Running](https://billgrant.io/2025/11/24/flask-setup-post/) - Basic Flask setup and data display
- [Visualizing Genre Connections](https://billgrant.io/2025/11/25/vis-js-blog-post/) - Graph visualization with Vis.js
- [Adding Bands to the Graph](https://billgrant.io/2025/11/26/bands-blog-post/) - Band nodes and primary genre connections
- [Interactive Expand/Collapse](https://billgrant.io/2025/11/26/phase-1-5-blog-post/) - Click to show/hide bands (Phase 1 complete)

**Phase 2 Posts:**
- [Database Integration](https://billgrant.io/2025/11/26/phase-2-blog-post/) - SQLAlchemy ORM and proper hierarchy (Phase 2 complete)

**Phase 3 Posts:**
- [CRUD Operations and Admin Panel](https://billgrant.io/2025/11/26/phase-3-blog-post/) - Full data management (Phase 3 complete)

**Phase 4 Posts:**
- [User Authentication and Authorization](https://billgrant.io/2025/11/26/phase-4-blog-post/) - Login and role-based access (Phase 4 complete)

**Phase 5 Posts:**
- [Production Deployment to GCP](https://billgrant.io/2025/11/26/phase-5-blog-post/) - Dockerization, Terraform, SSL, production infrastructure

**Current Phase:** Phase 6 - CI/CD and DevOps

All posts are tagged with [#music-graph](https://billgrant.io/tags/#music-graph).

---

*This is part of an ongoing series building a music genre graph application. Future posts will document each phase of development.*
