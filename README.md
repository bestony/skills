[![skills.sh](https://skills.sh/b/bestony/skills)](https://skills.sh/bestony/skills)

# bestony/skills

Reusable AI-agent skills packaged for the [skills.sh](https://skills.sh) ecosystem.
This repository collects focused, portable skills that can be installed into
compatible agents and invoked when their domain-specific workflows are needed.

## Installation

Install the collection with the skills.sh CLI:

```bash
npx skills add bestony/skills
```

## Available Skills

- `caprover-log-analysis`: Read, search, and analyze logs on CapRover and
  Docker Swarm servers, including app services, `captain-nginx`, and
  `captain-captain`.
- `performance-optimize`: Performance and high-concurrency tuning for Linux
  servers, Docker, Docker Swarm, CapRover, Nginx reverse proxies, and Go web
  services.

## Usage

After installation, invoke the skill from a compatible agent when you need its
workflow. For example:

```text
$caprover-log-analysis
$performance-optimize
```
