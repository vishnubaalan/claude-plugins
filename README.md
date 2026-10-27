# claude-plugins

Claude Code plugins for building Breezeware's React web apps.

## Plugins

| Plugin | Skill | What it does |
|---|---|---|
| `react-fe` | `vishnu-reactjs-frontend` | Builds screens, features, components, forms, dialogs, tables, API services, hooks and slices the way Breezeware's web apps are built: React 19, JavaScript, Vite, shadcn/ui + Tailwind, Redux Toolkit, Axios with Cognito auth, Formik + Yup |

## Install

In Claude Code:

```
/plugin marketplace add vishnubaalan/claude-plugins
/plugin install react-fe@vishnu-plugins
```

Restart Claude Code when asked. The skill is then listed as `react-fe:vishnu-reactjs-frontend`.

## Use

The skill loads by itself when you ask for frontend work in a Breezeware app ("create a form dialog for …", "add a service for …"). To be sure it is the one used, call it by name:

```
/react-fe:vishnu-reactjs-frontend add a People table with search and paging
```

## What the skill covers

- Project structure: `screens/` vs flat `features/`, where services, slices and hooks go, and how older apps (ChorusCare) differ
- API layer: singleton service classes on the global axios, `params` instead of hand-built query strings, `apiError`, the Spring `Page` contract
- State: a feature data hook per screen (`async`/`await`, a `live` flag, `refresh()`), Redux only for session and UI flags, passing data by props instead of Context
- Parallel loads with `Promise.allSettled`, and optimistic updates that roll back
- Formik + Yup forms, routing and roles, notifications, the four screen states
- Tailwind 3/4 tokens, dark mode, responsive layout, accessibility, dates and code style

The full guide is [`SKILL.md`](plugins/react-fe/skills/vishnu-reactjs-frontend/SKILL.md), with code recipes in [`references/component-patterns.md`](plugins/react-fe/skills/vishnu-reactjs-frontend/references/component-patterns.md).

## Update

```
/plugin marketplace update vishnu-plugins
```

Releases bump `version` in `plugins/react-fe/.claude-plugin/plugin.json` and are tagged (`v1.1.0`).
