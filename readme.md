# Common Github Actions

Sample usage:

```yaml
name: Lint and check lockfile

on:
  push:
  pull_request:

permissions:
  contents: read

jobs:
  linting:
    uses: nuhs-projects/common-actions/.github/workflows/lint-python.yml@main

  check-lockfile:
    uses: nuhs-projects/common-actions/.github/workflows/check-uv-lockfile.yml@main
    with:
      working_directory: backend
```

## Limitations

- Our Github Free plan [doesn't support] organization-level secrets and variables.
- Workflow files cannot [be in folders].

[doesn't support]: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-an-organization
[be in folders]: https://github.com/orgs/community/discussions/10773