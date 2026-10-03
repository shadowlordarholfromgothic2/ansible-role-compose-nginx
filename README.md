# `template_role` role

> **Using this template** — delete this block once the new role is set up.
>
> 1. Copy the directory (without `.git/` and `.ansible/`) and rename the role
>    everywhere. The name must match `^[a-z][a-z0-9_]*$`:
>
>    ```bash
>    grep -rlE 'template[_-]role' --exclude-dir=.git --exclude-dir=.ansible . \
>      | xargs sed -i -e 's/template_role/my_role/g' -e 's/template-role/my-role/g'
>    ```
>
> 2. Fill in `meta/main.yml` (description, platforms, tags) and
>    `requirements.yml` (the collections the role uses).
> 3. Replace the example config-file logic in `defaults/`, `tasks/`,
>    `templates/`, `handlers/` and `molecule/default/` with the real role.
>    Everything marked `TODO` needs attention.
> 4. Fill in the sections of this README.
> 5. On GitHub: use squash merges with the PR title as the commit message, and
>    optionally add a `RELEASE_PLEASE_TOKEN` secret (see [Releases](#releases)).
>
> Before the first push, run the same checks as CI (see [Testing](#testing)).

TODO: one paragraph on what the role does — and what it deliberately leaves
to other roles.

## Requirements

* ansible-core ≥ 2.14
* TODO: collections (also listed in [`requirements.yml`](requirements.yml)).
* TODO: what must already be on the target host. If the role checks for it
  rather than installing it, say so.
* `become: true` — TODO: say why.

## What the role does

1. **Validates input** — TODO: the checks and what they guard against.
2. TODO: one numbered step per section of [`tasks/main.yml`](tasks/main.yml).

On the managed host it leaves:

```text
/etc/template_role/
└── config.ini              # TODO: the files and directories the role manages
```

## Variables

See [`defaults/main.yml`](defaults/main.yml) for the full commented list.

| Variable                   | Default              | Purpose |
|----------------------------|----------------------|---------|
| `template_role_state`      | `present`            | `present` / `absent` |
| `template_role_config_dir` | `/etc/template_role` | Configuration directory on the host |
| `template_role_settings`   | `{}`                 | Key/value pairs written to `config.ini` |

TODO: list here any variables the role registers for later tasks.

## Usage

```yaml
# group_vars/all.yml
template_role_settings:
  log_level: info
```

```yaml
- hosts: all
  become: true
  roles:
    - role: template_role
```

### Removing

```bash
ansible-playbook playbook.yml -e template_role_state=absent
```

TODO: say what is removed and what is kept.

## Tags

| Tag      | Tasks |
|----------|-------|
| `always` | Input validation |
| `config` | TODO |

## Notes

* TODO: design decisions and pitfalls a user should know about — anything
  that would surprise someone reading the tasks for the first time.

## Testing

A Molecule scenario runs the role in a Debian 13 container, then checks
idempotence and TODO: what [`verify.yml`](molecule/default/verify.yml)
asserts. CI runs it, together with both linters, on every pull request:

```bash
pip install -r requirements-dev.txt     # same pinned versions as CI
yamllint --strict .
ansible-lint
molecule test
```

ansible-lint and Molecule install the collections and test roles from
[`requirements.yml`](requirements.yml) on their own.

## Releases

PRs are squash-merged, so the PR title becomes the commit on `main`. It must be
a [Conventional Commit](https://www.conventionalcommits.org/) (a check enforces
this), because it decides the next version and the changelog entry:

| PR title                                              | Next release | Changelog       |
|-------------------------------------------------------|--------------|-----------------|
| `feat!: …` or a `BREAKING CHANGE:` footer             | major        | ⚠ Breaking      |
| `feat: …`                                             | minor        | Features        |
| `fix:` / `perf:` / `revert:` / `docs: …`              | patch        | own section     |
| `ci:` / `test:` / `refactor:` / `build:` / `style:` / `chore: …` | none | not listed |

After each merge, [release-please](https://github.com/googleapis/release-please)
opens or updates a release PR with the version bump and the new `CHANGELOG.md`
entry. Merging that PR tags the release (e.g. `1.2.0`) and publishes a GitHub
release.

PRs opened with the default `GITHUB_TOKEN` do not trigger workflows, so the
release PR gets no CI checks. If branch protection requires them, add a
fine-grained PAT as the `RELEASE_PLEASE_TOKEN` secret.

## License

MIT
