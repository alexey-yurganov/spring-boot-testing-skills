# Spring Boot Skills

A collection of reusable skills for [Claude](https://claude.ai) and [OpenSkills](https://github.com/anthropics/skills) that provide specialized capabilities for software development, document generation, design, and more.

## What Are Skills?

Skills are structured instruction sets that teach Claude how to perform specific tasks. When a task matches a skill's description, Claude loads the skill's instructions and bundled resources to complete it more effectively.

## Usage

Load a skill when working on a matching task:

```bash
npx openskills read <skill-name>
```

Load multiple skills at once:

```bash
npx openskills read skill-one,skill-two
```

Each skill provides its own instructions, code templates, and bundled scripts under `references/`, `scripts/`, and `assets/` directories.

## Available Skills

| Skill | Description |
|-------|-------------|
| [spring-boot-testing](./spring-boot-testing/) | Writing unit and integration tests for Spring Boot applications (JUnit 5, Mockito, @WebMvcTest, @DataJpaTest, Testcontainers) |

## Project Structure

Each skill contains a `SKILL.md` with instructions and may include:

- `references/` — Detailed guides loaded as needed
- `scripts/` — Reusable utility scripts
- `assets/` — Templates, icons, fonts, and other resources
- `evals/` — Test prompts and evaluation data
