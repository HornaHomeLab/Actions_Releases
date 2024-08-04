# Actions_Releases
##### version: 1.0.0
GitHub reusable Workflows definition used to automate publishing releases in repositories.

Retrieve version from `README.md` in repository root, looking for phrase:

    ##### version:

Everything in same line after `:` will be considered as desired version

## Create Release process
1. **Check Version**:
   - Retrieves version from `README.md`
   - Checks if tag is not referenced to Full Release. 
      If it is, than job will fail to block `PR` from merging.

2. **Create Tag**:
   - Creates new tag if the one with retrieved version does not exist.
   - If tag for this version exists, then it updates tag reference

3. **Create Release**:
   - Takes `pre_release` flag as an input param.
   - Creates new Release if the one with retrieved version does not exist.
   - If Release exists updates it.

4. **Upload Assets**:
   - Takes `files_to_archive` and `archive_name` as an input param.
        `files_to_archive` is a `space` separated list of paths which should be included in archive.
   - Deletes existing asset
   - Creates new archive with files from `files_to_archive` list.
   - Uploads new archive to the created release.

## Remove Release process
1. **Check Version**:
   - Retrieves version from `README.md`
   - Checks if tag is not referenced to Full Release. 
      If it is, than job will fail to block any modifications on Full Release.

2. **Remove Release**:
   - Removes release if exists.

# Sample workflow 
Example Workflow to place in application repository to trigger Releases creation on `pull requests`.
It includes the deployment to dev environment using workflows from [Actions_Deployments](https://github.com/HornaHomeLab/Actions_Deployments).

When PR is opened, pre-release is created on each push. 
If the current version is overlapping with existing Full Release version, 
then Workflow with fail to block merge option (as a result pre-release will not be created).

On PR merge existing pre-release is updated with new assets and references and marked as Full Release.

If PR is closed without merge, than existing pre-release and version tag is removed.

> [!WARNING]  
> If PR containing conflicts is being closed, auto-cleanup will not work.
> No GitHub Actions can be started on PR with conflicts (current GitHub Actions limitation)

```YAML
name: CI/CD

on:
  pull_request:
    types: [ opened, synchronize, closed ]

permissions:
  contents: write

jobs:
  Pre:
    if: github.event.pull_request.merged == false && (github.event.action == 'opened' || github.event.action == 'synchronize')
    uses: HornaHomeLab/Actions_Releases/.github/workflows/Process_create_release.yml@main
    with:
      pre_release: true
      archive_name: "test"

  Full:
    if: github.event.pull_request.merged == true && github.event.action == 'closed'
    uses: HornaHomeLab/Actions_Releases/.github/workflows/Process_create_release.yml@main
    with:
      pre_release: false
      archive_name: "test"
      latest: true

  Deploy:
    needs: Full
    uses: HornaHomeLab/Actions_Deployments/.github/workflows/Run_docker_compose.yml@main
    with:
      environment: "development"
      version: ${{ needs.Full.outputs.released_version }}
    secrets: inherit

  Remove:
    if: github.event.pull_request.merged == false && github.event.action == 'closed'
    uses: HornaHomeLab/Actions_Releases/.github/workflows/Process_remove_release.yml@main

```
