# Packaging POC

> The purpose of this repo is to demonstrate how to automate the build and release of an internal npm package.

[Go to package](https://github.com/real-io/github-actions-playground/pkgs/npm/github-actions-playground)

## How do we do it?

1. Setup the package to be published to the GitHub package registry:

   - At your account level, create a personal access token with the `write:packages` scope if you don't already have one.
   - Add below to your `package.json`:
     ```json
     "publishConfig": {
        "@real-io:registry": "https://npm.pkg.github.com/"
      },
     ```

1. Set up the workflow in `.github/workflows` to trigger when a push is made to main:

```yml
name: Publish Package to Github Packages
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
    #
    # MORE STEPS GO HERE that are detailed below...
    #
```

3. Change the name in `package.json` to include the scope because the package name must be scoped to the organization:

```json
{
  "name": "@real-io/{name-of-package}",
  "version": "x.x.x",
  // ...
}
```

## Workflow Explanation

This GitHub Actions workflow is designed to automate the process of publishing a package to GitHub Packages and creating a release. The workflow consists of two jobs: `test-build` and `release-publish`. Below is an explanation of the core steps in each job:

## Job: `test-build`

### Trigger

This job is triggered on pushes to the `main` branch.

### Steps

1. **Checkout:** The repository is checked out using the `actions/checkout` action.

```yml
- name: Checkout
  uses: actions/checkout@v4
```

2. **Setup Node:** Node.js is set up using the `actions/setup-node` action.

```yml
- name: Setup Node
  uses: actions/setup-node@v3
  with:
    node-version: '20.x'
```

3. **Install and Check TypeScript:** TypeScript is installed globally, and its version is checked.

```yml
- name: Install and Check TypeScript
  run: |
    echo "Install TypeScript:"
    npm install -g typescript
    echo "Check TypeScript version:"
    tsc --version
```

4. **Install Dependencies:** npm dependencies are installed.

```yml
- name: Install Dependencies
  run: npm install
```

5. **Run Tests:** npm tests are executed.

```yml
- name: Run Tests
  run: npm test
```

6. **Bump Version:** The package version is bumped using the [phips28/gh-action-bump-version](https://github.com/marketplace/actions/update-version-in-package-json) action.

```yml
- name: Bump Version
  uses: 'phips28/gh-action-bump-version@master'
  with:
    tag-prefix: ''
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }
```

> [!IMPORTANT]  
> Based on the commit messages, increment the version from the latest release.
> 1. If the string **"major"** is found anywhere in any of the commit messages or descriptions the major version will be incremented.
> 2. If includes **"minor"** then the minor version will be increased.
> 3. If includes **"patch"** or NONE of the above then the patch version will be increased.

7. **Check Package.json:** Contents of the `package.json` file are displayed.

```yml
- name: Check Package.json
  run: cat ./package.json
```

8. **Check Local Files:** Contents of the root directory are listed.

```yml
- name: Check Local Files
  run: ls -a
```

9. **Get NPM Version:** The current version of the package is retrieved using the `npm-get-version-action`.

```yml
- name: Get NPM Version
  id: package-version
  uses: martinbeentjes/npm-get-version-action@v1.3.1
```

10. **Upload Package Files:** Relevant package files (lib directory, LICENSE, README.md, package.json) are uploaded as artifacts using the `actions/upload-artifact` action.

```yml
- name: Upload Package Files
  uses: actions/upload-artifact@v4
  with:
    # Name of the artifact to upload.
    # Optional. Default is 'artifact'
    name: package-files

    # A file, directory or wildcard pattern that describes what to upload
    # Required.
    path: |
      index.js
      LICENSE
      README.md
      package.json

    # The desired behavior if no files are found using the provided path.
    # Available Options:
    #   warn: Output a warning but do not fail the action
    #   error: Fail the action with an error message
    #   ignore: Do not output any warnings or errors, the action does not fail
    # Optional. Default is 'warn'
    if-no-files-found: error
```

### Outputs

- `version`: The version of the package is output for later use.

```yml
outputs:
  version: ${{ steps.package-version.outputs.current-version}}
```

## Job: `release-publish`

### Trigger

This job is triggered after the completion of the `test-build` job.

### Steps

1. **Download Package Files:** Artifact files (lib directory, LICENSE, README.md, package.json) are downloaded using the `actions/download-artifact` action.

```yml
- name: Download Package Files
  uses: actions/download-artifact@v4
  with:
    # Name of the artifact to download.
    # If unspecified, all artifacts for the run are downloaded.
    # Optional.
    name: package-files
```

2. **Create Release:** A GitHub release is created using the `actions/create-release` action, utilizing the version obtained from the `test-build` job.

```yml
- name: Create Release
  id: create_release_id
  uses: actions/create-release@v1
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  with:
    tag_name: ${{ needs.test-build.outputs.version }}
    release_name: Release ${{ needs.test-build.outputs.version }}
```

3. **Setup .npmrc:** A `.npmrc` file is set up to configure authentication for publishing to GitHub Packages.

```yml
# Setup .npmrc file to publish to Github Packages
- name: Setup .npmrc
  run: |
    echo @real-io:https://npm.pkg.github.com/ > .npmrc
    echo '//npm.pkg.github.com/:_authToken=${NPM_TOKEN}' >> .npmrc
```

4. **Publish Package:** The package is published to GitHub Packages using the `npm publish` command, with the authentication token provided.

```yml
- name: Publish Package
  run: npm publish
  env:
    NPM_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Note

Make sure to set up the necessary secrets, such as `GITHUB_TOKEN` for GitHub API access and `NPM_TOKEN` for npm registry authentication.

This workflow is designed for projects using npm packages and GitHub Packages for package hosting. Customize it according to your project's specific needs.

View the full workflow implementation [here](.github/workflows/npmPublish.yml).

---
