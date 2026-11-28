# Crux migration check

A GitHub Action that checks a pull request's SQL Server migrations **by running them**, before they are merged.

On every pull request it:

1. starts a throwaway SQL Server in a container on the runner,
2. builds a database there from the migrations on the branch the pull request will merge into,
3. runs the pull request's new migration scripts in it, then rolls them back,
4. comments on the pull request with what happened, and marks the check failed when something would break.

It reports:

- **Errors SQL Server gives**, on the script's own line: a column that does not exist, a NOT NULL column added to a table that has rows, a typo.
- **Code the change breaks**: every view, procedure, function and trigger that no longer compiles after it. On the open-source Bitwarden server, renaming one column broke 31 objects; reading the script alone finds 6.
- **What reading the scripts finds**: a column dropped or renamed while code still uses it, an edit to a migration that has already run, scripts out of order, a view that selects `*` from a changed table, a column narrowed.
- **The schema before and after**, as a diff.

Findings also appear beside the lines of the pull request's diff, and in the run's summary.

It is built on [Crux](https://github.com/ZEUSXXIV/crux-downloads), which explains why every table and column in a database exists, from its migrations, commits and pull requests.

## Use it

Add `.github/workflows/crux.yml` to your repository:

```yaml
name: Crux
on: pull_request

permissions:
  contents: read
  pull-requests: write   # to comment on the pull request

jobs:
  migrations:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0     # the check reads both sides from git, back to where the branch started
      - uses: ZEUSXXIV/crux-action@v1
```

To stop a breaking migration from being merged, make the **Crux / migrations** check required in the branch's protection rules.

## Inputs

| Input | Default | |
|---|---|---|
| `migrations` | found automatically | The migrations folder, relative to the repository. Several can be separated with semicolons. |
| `base` | the pull request's base branch | The branch or commit to compare with. On a push, the previous commit. |
| `fail-on` | `error` | Fail the check on `error`, `warning`, or `never`. |
| `comment` | `true` | Comment on the pull request, updating the same comment on each push. |
| `sandbox` | `true` | Run the new scripts in a throwaway SQL Server. `false` only reads them. |
| `version` | `latest` | The Crux version to use, such as `0.5.3`. |
| `sql-server-image` | `mcr.microsoft.com/mssql/server:2022-latest` | The SQL Server container. |

It needs a Linux runner (`ubuntu-latest`), since SQL Server runs in a Linux container.

## What it reads and writes

- Everything happens on your runner. Your code and scripts are not sent anywhere else, and no model or AI service is used.
- The SQL Server is created for the job, with a random password, and removed at its end. Your own databases are never touched.
- Crux runs the scripts as a database user that SQL Server keeps inside the throwaway database, refuses scripts that try to reach outside it, and rolls every run back.
- The only thing written to GitHub is the comment, with the token the job already has.

Pull requests from forks get a read-only token, so they get no comment; the report is still in the run's summary, and the check still passes or fails.

## Supported

SQL Server migrations in the common layouts: Flyway (`V1__name.sql`, with placeholders), DbUp and other numbered or dated scripts. PostgreSQL migrations are read and checked, but not yet run.

## License

The files in this repository are under the MIT license. Crux itself, which the action downloads from its releases, is
proprietary and free to use while it is a preview, under the [Crux End-User Licence Agreement](https://github.com/ZEUSXXIV/crux-downloads/blob/main/EULA.md).
