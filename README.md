# Nemme Agent Skills

[Claude Code](https://claude.ai/code) skills for integrating with Nemme products.

## Available Skills

| Skill | Description |
|---|---|
| [nemme-sdk](./nemme-sdk/) | Integrate the `@nemme/js-sdk` for event tracking, page views, forms, and delivery triggers |

## Installation

```bash
npx skills add nemme/skills
```

Or install globally (available in all projects):

```bash
npx skills add nemme/skills -g
```

## Usage

Once installed, skills activate automatically when relevant — for example, when Claude sees `@nemme/js-sdk` imports or you ask about Nemme tracking. You can also invoke manually via slash command:

```
/nemme-sdk
```

## Publishing

Skills are maintained in the [nemme monorepo](https://github.com/nemme/nemme) under `skills/` and published to [nemme/skills](https://github.com/nemme/skills).

To publish an update:

```bash
cp -r skills/ /path/to/nemme-skills/
cd /path/to/nemme-skills
git add . && git commit -m "Update skills"
git push
```

Users can update installed skills with:

```bash
npx skills update
```
