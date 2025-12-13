# Rain World Architecture Analysis

Comprehensive analysis of Rain World's code architecture, design patterns, and potential improvements.

## View Documentation

### Option 1: GitHub Pages (Once Enabled)

After enabling GitHub Pages:
1. Go to repository Settings → Pages
2. Set source to "Deploy from a branch"
3. Select branch: `main` (or `master`)
4. Select folder: `/docs`
5. Save

Documentation will be available at: `https://<username>.github.io/<repo-name>/`

### Option 2: Local Preview

Serve the docs locally with any web server:

```bash
# Using Python 3
cd docs
python3 -m http.server 3000

# Using Python 2
cd docs
python -m SimpleHTTPServer 3000

# Using Node.js (if you have npx)
cd docs
npx serve
```

Then open: http://localhost:3000

### Option 3: VS Code Live Server

1. Install "Live Server" extension in VS Code
2. Open `docs/index.html`
3. Right-click → "Open with Live Server"

## Documentation Structure

```
docs/
├── README.md                               # Home page
├── architecture/
│   ├── overview.md                         # Architecture overview
│   ├── core-patterns.md                    # Key patterns explained
│   ├── problems/
│   │   ├── singleton-coupling.md           # Singleton issues
│   │   └── inheritance-rigidity.md         # Inheritance issues
│   ├── improvements/
│   │   ├── ecs-migration.md                # ECS proposal
│   │   └── dependency-injection.md         # DI proposal
│   └── analysis/
│       └── 2025-12-14-review.md            # Full analysis
```

## What's Inside

**Current Architecture**:
- Dual LOD system (Abstract vs Realized entities)
- ProcessManager state machine
- Room-based world streaming
- BodyChunks physics

**Problems Identified**:
- Singleton coupling (global state)
- Deep inheritance hierarchies
- LOD transition data loss

**Proposed Solutions**:
- Entity Component System (ECS)
- Dependency Injection (DI)
- Seamless LOD transitions

## Quick Links

Once docs are served:
- [Architecture Overview](docs/architecture/overview.md)
- [Singleton Coupling Problem](docs/architecture/problems/singleton-coupling.md)
- [ECS Migration Proposal](docs/architecture/improvements/ecs-migration.md)

## Source

Analysis based on [Rain World Modding Wiki](https://rainworldmodding.miraheze.org/wiki/Rain_World_Code_Structure)
