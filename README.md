# Shared-Workflows

Reusable workflows for shared CI/CD usage across projects.

## Available reusable workflows

- Vue frontend: `/.github/workflows/reusable-vue.yml`
- PHP backend: `/.github/workflows/reusable-php.yml`

## Vue frontend workflow usage

```yaml
name: Frontend CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  vue-ci:
    uses: mentrd/Shared-Workflows/.github/workflows/reusable-vue.yml@main
    with:
      node-version: "20"
      working-directory: "."
      install-command: "npm ci"
      build-command: "npm run build --if-present"
      test-command: "npm test --if-present"
      run-build: true
      run-test: true
```

## PHP backend workflow usage

```yaml
name: Backend CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  php-ci:
    uses: mentrd/Shared-Workflows/.github/workflows/reusable-php.yml@main
    with:
      php-version: "8.3"
      working-directory: "."
      dependency-command: "composer install --no-interaction --no-progress --prefer-dist"
      test-command: "composer test"
      run-test: true
```
