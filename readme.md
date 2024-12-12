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
  lint:
    uses: nuhs-projects/common-actions/.github/workflows/python.yml@main
    with:
      working_directory: backend
```

## Notes

- Naming convention: `${language}-${action-name}`. For actions applicable to all languages, omit `${language}`.
- Our Github Free plan [doesn't support] organization-level secrets and variables in private repositories.
- Called workflows only see the caller workflow repository's variables and secrets.
- Workflow files cannot [be in folders].

[doesn't support]: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-an-organization
[be in folders]: https://github.com/orgs/community/discussions/10773
