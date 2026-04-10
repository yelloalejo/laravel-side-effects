# Laravel Side Effects

An agent skill that thoroughly analyzes side effects and collateral impact of proposed changes in Laravel projects before executing them.

## What it does

Before any code change is executed, this skill performs an exhaustive impact analysis covering:

- **Eloquent Events & Observers** — Detects when a new `save()`, `update()`, or `delete()` could trigger observers, events, or listeners in unexpected contexts
- **JsonResource Mismatch** — Catches silent bugs where a renamed/removed model attribute causes a Resource to return `null` instead of erroring
- **Full Effect Chain** — Traces the complete chain: `save()` → observer → event → listener → notification → webhook
- **Queued Jobs** — Flags models with `SerializesModels` that may break if the model structure changes while jobs are in queue
- **Hidden Trait Logic** — Identifies traits with boot-time event registration that fire invisibly
- **Scopes & Accessors** — Warns when global scopes affect all queries or when accessor changes silently alter computed values
- **Relations** — Detects renamed relations that break `whenLoaded()` in Resources
- **Mass Assignment** — Verifies `$fillable`/`$guarded` stay in sync with model changes
- **Middleware & Policies** — Checks for unintended access changes
- **Database Migrations** — Validates rollback support, data integrity, and critical table safety
- **Custom Project Rules** — Configurable per-project rules (e.g., "always verify financial precision when touching transaction models")

## Installation

### Via skills.sh (Claude Code, Cursor, etc.)

```bash
npx skills add yelloalejo/laravel-side-effects
```

### Via Craft Agent

Copy the `skills/laravel-side-effects/` folder to:
```
~/.craft-agent/workspaces/{your-workspace}/skills/laravel-side-effects/
```

### Manual

Copy `skills/laravel-side-effects/SKILL.md` to your agent's skills directory.

## Configuration

After installing, create your project-specific config:

```bash
cp config.example.json config.json
```

Edit `config.json` with your project's details:

| Field | Description |
|-------|-------------|
| `criticalModels` | Models that require extra scrutiny (with related Resources and Observers) |
| `criticalTables` | Database tables that auto-elevate severity |
| `customRules` | Project-specific rules that are always checked (e.g., financial precision) |
| `directories` | Override paths if your project doesn't follow standard Laravel conventions |
| `notifications` | Toggle specific warnings (new save, relation change, missing migration down, etc.) |
| `ignorePaths` | Directories to skip in analysis |

## Usage

Invoke the skill before executing a plan or making structural changes:

```
/laravel-side-effects
```

Or mention it in conversation:

```
[skill:laravel-side-effects] Review the side effects of this change
```

The skill auto-suggests when working with `.php` files in `app/`, `routes/`, `database/migrations/`, or `config/`.

## Output Format

The analysis produces a structured report:

- **Affected models/tables** (flagged if critical)
- **Effect chain diagram** (change → event → listener → action)
- **Custom rules evaluation**
- **Findings classified by severity**: Critical, High, Medium, Low
- **Mitigations** for each critical/high finding
- **Conclusion** with go/no-go recommendation

## License

MIT
