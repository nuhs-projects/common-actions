# Common Github Actions

Add automated linting, testing, secret scanning and deployment to your repository.

Lint, scan for secrets and build the `test` stage of a Dockerfile:

```yaml
# .github/workflows/lint-and-test.yml
name: Lint and Test
on:
  pull_request:
    branches: [main]
  workflow_call:

permissions:
  contents: read
  id-token: write

jobs:
  lint:
    uses: nuhs-projects/common-actions/.github/workflows/python-lint.yml@v0.7

  test:
    uses: nuhs-projects/common-actions/.github/workflows/build.yml@v0.7
    needs: lint
    with:
      ecr_repository: namespace/repo # Replace
      target: test # Replace as necessary
      push: false
      iam_role: arn:aws:iam::123456789012:role/ExampleRole # Replace
      aws_region: ap-southeast-1
```

Build and push the image to the Dev AWS ECR, generate K8S configs then deploy to a K8S cluster:

```yaml
# .github/workflows/build-and-deploy.yml
name: Build, Push to ECR and Deploy to `dev` K8S Cluster
on:
  push:
    branches:
      - main
    paths-ignore:
      - "**.md"

permissions:
  contents: write
  id-token: write

jobs:
  get-commit-metadata:
    uses: nuhs-projects/common-actions/.github/workflows/commit-metadata.yml@v0.7

  build:
    uses: nuhs-projects/common-actions/.github/workflows/build.yml@v0.7
    needs: get-commit-metadata
    with:
      ecr_repository: chatbot/russell-gpt-web # Change this
      target: deployment
      push: true
      iam_role: ${{ vars.IAM_ROLE }}
      aws_region: ${{ vars.AWS_REGION }}
      build_args: |
        COMMIT_SHA=${{ needs.get-commit-metadata.outputs.sha_short }}
        COMMIT_TIMESTAMP=${{ needs.get-commit-metadata.outputs.timestamp }}
      # You can add extra tags, e.g. the name of a pushed tag
      extra_tags: |
        ${{ github.ref_name }}

  generate-k8s:
    uses: nuhs-projects/common-actions/.github/workflows/generate-k8s.yml@v0.7
    needs: build
    with:
      dev_deployment_tags: | # Change as necessary
        deployment=russell-gpt,tag=${{ needs.build.outputs.image_tag }}

      # To generate image tags for staging/production:
      staging_deployment_tags: |
        deployment=russell-gpt,tag=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-repo:example-tag
      production_deployment_tags: |
        deployment=russell-gpt,tag=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-repo:example-tag

  deploy:
    uses: nuhs-projects/common-actions/.github/workflows/dev-deploy.yml@v0.7
    needs: generate-k8s
    with:
      iam_role: ${{ vars.IAM_ROLE }}
      aws_region: ${{ vars.AWS_REGION }}
      k8s_cluster_name: dev
```

## `python-lint.yaml`

- Linting via Ruff, using the project's `ruff.toml` if it exists
- Secret scanning via Trufflehog
- Optionally, lints using the `ruff.toml` from this repo.
- Optionally, checks the uv lockfile

Note: This uses `ruff` from the project's dev dependencies.

## `build.yml`

Builds a Dockerfile (or a stage of it), caching via AWS ECR. Optionally, pushes the built image to ECR.

## `generate-k8s.yml` action

This action uses `kompose` to generate a base K8S yaml from a Docker `compose.yml` and then `kustomize` to generate environment-specific K8S yamls for specific environments.

Your repository needs to have a `deployment` folder in the root (name is customizable), containing:

- `compose.yml`

  - The service to be exposed via an Ingress must contain the following label:
    ```yaml
    labels:
      kompose.service.expose: kompose.host
    ```
  - Note that the names of the services will become the names of the deployments.

- A `base` folder containing a `kustomization.yaml`, with the following (minimally):

  ```yaml
  # Configuration here is applied to all kustomizations which inherit this
  # e.g. overlays/{dev,staging,production}/kuztomization.yaml
  apiVersion: kustomize.config.k8s.io/v1beta1
  kind: Kustomization

  resources:
    - base.yaml
  ```

- Kustomize overlays (all are optional):

  - `overlays/dev/kustomization.yaml`
  - `overlays/staging/kustomization.yaml`
  - `overlays/production/kustomization.yaml`
  - Note:

    - In the `resources` section, you should include `base.yaml` as follows:
      ```yaml
      resources:
        - ../../base
      ```

  - For an example of Kustomize overlays, see the [medivoice repo].

## `dev-deploy.yml`

Checks and applies a `dev.yaml` K8S config to a cluster.

## Notes

- Tests are run by building the `test` stage of a Dockerfile, as shown above.
- Naming convention: `${language}-${workflow-name}`.
  - For actions applicable to all languages, omit `${language}`.
- Our Github Free plan [doesn't support] organization-level secrets and variables in private repositories.
- Called workflows only see the caller workflow repository's variables and secrets.
- Environment variables set in the `env` context, defined in the called workflow, are [not accessible] in the `env` context of the caller workflow. Use `vars` instead.
- Workflow files cannot [be in folders].

[doesn't support]: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/store-information-in-variables#creating-configuration-variables-for-an-organization
[be in folders]: https://github.com/orgs/community/discussions/10773
[medivoice repo]: https://github.com/nuhs-projects/medivoice/tree/main/deployment/overlays
[not accessible]: https://docs.github.com/en/actions/sharing-automations/reusing-workflows#limitations
