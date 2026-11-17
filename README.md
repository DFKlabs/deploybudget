<h1 align="center">deploybudget</h1>

<p align="center">
  <strong>Stop Vercel from building every branch you push.</strong>
  <br>
  Fixes <em>&quot;Deployment rate limited &mdash; retry in 24 hours&quot;</em> and stops paying for builds nobody opens.
</p>

<p align="center">
  <a href="https://github.com/dfklabs/deploybudget/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/dfklabs/deploybudget/actions/workflows/ci.yml/badge.svg"></a>
  <img alt="Zero dependencies" src="https://img.shields.io/badge/dependencies-0-brightgreen">
  <img alt="Node 18 or newer" src="https://img.shields.io/badge/node-%E2%89%A5%2018-brightgreen">
  <a href="#license"><img alt="Apache-2.0, templates CC0" src="https://img.shields.io/badge/license-Apache--2.0%20%C2%B7%20templates%20CC0-blue"></a>
</p>

<p align="center">
  <a href="#the-fix">The fix</a>
  &nbsp;&middot;&nbsp;
  <a href="#install">Install</a>
  &nbsp;&middot;&nbsp;
  <a href="#every-project-at-once">Every project at once</a>
  &nbsp;&middot;&nbsp;
  <a href="#faq">FAQ</a>
</p>

---

## The problem

By default, Vercel builds **every branch you push**: `feature/login`, `fix/typo`,
that experiment from last Tuesday. Each push is a full build.

- On the **Hobby** plan, that is how you end up locked out:

  ```
  Deployment rate limited — retry in 24 hours
  ```

- On **Pro**, build minutes are metered, so every one of those builds is on your bill.

deploybudget is a drop-in `vercel.json` (and a small script to apply it to all
your projects) that keeps automatic deployments for your production branch only.

---

## The fix

```json
{
  "$schema": "https://openapi.vercel.sh/vercel.json",
  "git": {
    "deploymentEnabled": {
      "*": false,
      "main": true
    }
  }
}
```

Deny every branch with the `*` wildcard, then allow production back. When a branch
matches several rules, **one `true` is enough to deploy**: `main` matches both
rules and deploys; every other branch matches only `*` and doesn't.

### Why the obvious version doesn't work

Most people try this first:

```jsonc
{
  "git": {
    "deploymentEnabled": {
      "main": true,
      "develop": false,
      "staging": false
    }
  }
}
```

Vercel treats **any branch you don't name as `true`**. `develop` and `staging`
stop, but `feature/login`, `fix/typo` and every branch you create next month
keep deploying.

---

## Install

Pick whichever suits you. All three end with the same `vercel.json`.

### 1. Ask Claude

Paste this into [Claude Code](https://claude.com/claude-code) inside your project:

```text
Add deploybudget (github.com/dfklabs/deploybudget) to this project: detect my production branch, show me the vercel.json change first, then apply it so only production deploys automatically.
```

### 2. Install the Claude Code plugin

```bash
claude plugin marketplace add dfklabs/deploybudget
claude plugin install deploybudget@deploybudget
```

From then on, say *"my Vercel quota is maxed out"* in any project. Claude
detects your production branch, shows you the change and applies it.

### 3. By hand

Download the file into the root of your project, change `main` if your
production branch has another name, then commit and push:

```bash
curl -O https://raw.githubusercontent.com/dfklabs/deploybudget/main/vercel.json
```

Already have a `vercel.json`? Copy only the `git` block into it, or use the
script below, which merges instead of overwriting.

---

## Every project at once

`apply.mjs` needs Node 18 or newer and nothing else. It finds every project under
a folder, merges the `git` block into each existing `vercel.json`, and keeps your
formatting.

```bash
git clone https://github.com/dfklabs/deploybudget
cd deploybudget

# Preview the changes without writing anything
node apply.mjs --scan ~/code --dry-run

# Apply them, keeping a .bak copy of every file it changes
node apply.mjs --scan ~/code --backup

# Production is "master", and "staging" should keep deploying too
node apply.mjs --scan ~/code --branch master --allow staging
```

| Option | What it does |
| --- | --- |
| `--scan <path>` | Apply to every project found under `<path>` |
| `--branch <name>` | Production branch that keeps deploying (default: `main`) |
| `--allow <name>` | Another branch that keeps deploying (repeatable) |
| `--depth <n>` | How deep to look for projects (default: `3`) |
| `--force` | Remove existing rules that still let other branches deploy |
| `--dry-run` | Show what would change and write nothing |
| `--backup` | Save `vercel.json.bak` before changing a file |

A folder counts as a project when it has a `package.json` or a `vercel.json`.
`node_modules`, `.git`, `.next`, `dist` and similar folders are skipped. Running
it twice is safe: the second run reports `already production-only`.

### Ready-made variants

| File | Deploys automatically |
| --- | --- |
| [`templates/production-only.json`](./templates/production-only.json) | `main` only |
| [`templates/master-branch.json`](./templates/master-branch.json) | `master` only |
| [`templates/production-and-staging.json`](./templates/production-and-staging.json) | `main` and `staging` |
| [`templates/disable-all.json`](./templates/disable-all.json) | Nothing; deploy from the CLI yourself |

---

## What changes, and what doesn't

- **Branch pushes stop deploying**, except the branches you allow. That is where
  the quota and the build minutes go.
- **Pull request previews stop too.** Previews are triggered by branch pushes, so
  `"*": false` blocks them as well. If you want previews on pull requests and
  nothing else, skip this file and use **Settings → Git** in the Vercel
  dashboard, which has a separate switch for that. `vercel.json` can't express
  "previews on pull requests only".
- **Manual deploys still work.** `vercel --prod` and the dashboard's Redeploy
  button are unaffected, so you always have a way out.
- **Your build settings are untouched.** The template has no `buildCommand`,
  `installCommand` or `devCommand`, so it never overrides the framework preset
  Vercel detected or breaks a pnpm, yarn or bun project.

### Why not just use the dashboard?

You can: **Settings → Git** covers most of this, one project at a time.

|  | Dashboard | `vercel.json` |
| --- | --- | --- |
| In version control | No | Yes |
| Stays put when someone else changes settings | No | Yes |
| Carries over to forks and new clones | No | Yes |
| Branch patterns | Limited | Full [minimatch](https://github.com/isaacs/minimatch) |
| Apply to many projects | One by one | One command |

`vercel.json` wins once you have more than a couple of projects. The dashboard
wins for the one thing only it can do: previews on pull requests only.

---

## FAQ

**Will this lift a rate limit I've already hit?**
No. The 24-hour window has to run out on its own. This stops the next one.

**Does a blocked branch still cost a build?**
No. Vercel reads `vercel.json` from the pushed commit before building and skips
the build entirely.

**My monorepo has several Vercel projects.**
Each Vercel project reads the `vercel.json` in its own root directory. Put one in
each root; `apply.mjs --scan` finds them all.

**A branch I blocked is still deploying.**
Some other rule matches it with `true`, and one `true` wins over any number of
`false`. Run `node apply.mjs <project> --force` to remove the conflicting rules.

---

## Contributing

No dependencies and no build step; Node 18 or newer is all you need.

```bash
npm test        # runs the real CLI against the fixtures in test/
npm run verify  # release checks: versions, templates, licences, packaging
```

Both run on Node 18, 20 and 22 on every push and pull request. Issues and pull
requests are welcome at
[github.com/dfklabs/deploybudget/issues](https://github.com/dfklabs/deploybudget/issues).

If deploybudget saved you a quota or a bill, a star helps other people find it.

---

## License

| Part | License | In short |
| --- | --- | --- |
| `apply.mjs` and the docs | Apache-2.0 | Use it anywhere, commercially too. Keep the notice and mark your changes. |
| `vercel.json`, `templates/*.json` | CC0-1.0 | Public domain. Paste them into your projects, no attribution needed. |

See [LICENSE](./LICENSE), [NOTICE](./NOTICE) and [templates/LICENSE](./templates/LICENSE).
"dfklabs" is a trademark of dfklabs and is not covered by these licenses.

Made by [dfklabs](https://dfklabs.com).
