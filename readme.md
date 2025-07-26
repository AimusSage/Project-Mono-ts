# Personal Monorepo for typescript

This is my personal monorepo for bringing together experiments, tooling, and learning projects. Over time it will come to include small utilities, automation scripts, Docker/container setups, and development notes. The goal is to consolidate ongoing work in one place and track personal growth across multiple areas of interest.


## Structure

```
.
├── cli-tools/         # Command-line utilities and scripts
├── containers/        # Custom Dockerfiles and images
├── docker-compose/    # Docker Compose service setups
├── homelab/           # Homelab-related configs and notes
├── notes/             # Markdown-based technical notes
├── playground/        # Free-form experiments and prototypes
├── scripts/           # Shell scripts for automation
├── utils/             # Reusable TypeScript helpers
```

## Getting Started

Install dependencies:

```bash
npm install
```

Run tests:

```bash
npm run test
```

Start a script:

```bash
npm run start
```

## Tech Stack

- **TypeScript** for all development logic  
- **Vitest** for unit testing  
- **ts-node** for executing scripts  
- **Docker / Docker Compose** for container management  
- **Shell scripting** for system-level automation

## Guidelines

- Keep each tool or utility self-contained in its own folder  
- Include a `README.md` in any folder with non-trivial logic or configuration  
- Avoid including sensitive data; use `.env.example` where needed  

## License

MIT