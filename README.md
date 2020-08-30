# Power Apps Extract Solution

This Action exports the specified solution in the source environment, extract and decomposes them them into individual components and commits them into the repo. This Action can also build the Managed and Unmanaged Solutions from the individual components in the repo, and publish these file to Release as assets.

## Inputs

| Input                | Required | Default | Description                                                                                                                                                                                                                                                                                                           |
| -------------------- | -------- | ------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| token                | ✅       | -       | [GitHub Personal Access Token](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token). This is required for downloading the solution files from Releases.                                                                                                                       |
| solutionName         | ✅       | -       | The Unique Name of the solution. This is used in Solution Upgrade step, if you are deploying a Managed Solution.                                                                                                                                                                                                      |
| sourceEnvironmentUrl | ✅       | -       | Environment URL from where the Solution will be downloaded from.                                                                                                                                                                                                                                                      |
| applicationId        | ✅       | -       | Application Id that will be used to connect to the Power Apps environment.Refer [Scott Durow's video](https://www.youtube.com/watch?v=Td7Bk3IXJ9s) if you do not know how to set Application Registration and scopes in Azure to facilitate this.                                                                     |
| applicationSecret    | ✅       | -       | Secret that will be used in the Connection String for authenticating to Power Apps. Use GitHub Secrets on your repo to store this value. Refer to [Using Encrypted Secrets](https://docs.github.com/en/actions/configuring-and-managing-workflows/                                                                    | creating-and-storing-encrypted-secrets#using-encrypted-secrets-in-a-workflow) on how to use this in your workflow. |
| releaseSolution      | ✅       | false   | By default, this Action does not publish the Managed and Unmanaged solutions into Releases. Set this to true, if you would like to do so. You need to publish the release with the solution files, if you are using the [Power Apps Deploy Solution](https://github.com/rajyraman/powerapps-deploy-solution/) Action. |

## Outputs

| Output                      | Description                                                                                               |
| --------------------------- | --------------------------------------------------------------------------------------------------------- |
| solutionVersion             | Automatically generate Solution version number. The format of the version is yyyy.MM.dd.action_run_number |
| solutionFileNameWithVersion | Solution File Name with the version number.                                                               |

## Example

Below is an example on how to use [Power Apps Solution Extract](https://github.com/rajyraman/powerapps-solution-extract/) and [Power Apps Deploy Solution](https://github.com/rajyraman/powerapps-deploy-solution/) in your Workflow.

```yaml
# This workflow is run only manually, as it uses a workflow_dispatch. It is recommended to use cron or push into main branch to trigger this. Since this workflow also commits into the repo, use ignore tags to prevent infinite loop. Refer https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#example-ignoring-branches-and-tags

name: extract-and-deploy
on:
  workflow_dispatch:
    inputs:
      solutionName:
        description: "Solution Name"
        required: true
        default: "GitHubSolution"
      sourceEnvironmentUrl:
        description: "Source Environment URL"
        required: true
        default: "https://xxxx.crm.dynamics.com"
      targetEnvironmentUrl:
        description: "Target Environment URL"
        required: true
        default: "https://xxxx.crm.dynamics.com"
      release:
        description: "Release?"
        required: true
        default: true
      managed:
        description: "Managed?"
        required: true
        default: true
      debug:
        description: "Debug?"
        required: true
        default: "true"

jobs:
  build:
    # The type of runner that the job will run on. This needs to be Windows runner.
    runs-on: windows-latest

    steps:
      - uses: actions/checkout@v2

      - name: Dump GitHub context
        if: github.event.inputs.debug == 'true'
        env:
          GITHUB_CONTEXT: ${{ toJson(github) }}
        run: echo "$GITHUB_CONTEXT"

      - name: Extract Solution
        id: extract-solution
        uses: rajyraman/powerapps-solution-extract@v1.1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          solutionName: ${{ github.event.inputs.solutionName }}
          sourceEnvironmentUrl: ${{ github.event.inputs.sourceEnvironmentUrl }}
          applicationId: ${{ secrets.APPLICATION_ID }}
          applicationSecret: ${{ secrets.APPLICATION_SECRET }}
          releaseSolution: ${{ github.event.inputs.release }}

      - name: Deploy Solution
        id: deploy-solution
        uses: rajyraman/powerapps-deploy-solution@v1.1
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          solutionName: ${{ github.event.inputs.solutionName }}
          targetEnvironmentUrl: ${{ github.event.inputs.targetEnvironmentUrl }}
          applicationId: ${{ secrets.APPLICATION_ID }}
          applicationSecret: ${{ secrets.APPLICATION_SECRET }}
          tag: ${{ steps.extract-solution.outputs.solutionVersion }}
          managed: ${{ github.event.inputs.managed }}


        # This step below can be removed, if you do not want to send a notification to Teams about this solution deployment.
      - name: Notify Teams
        uses: fjogeleit/http-request-action@v1.4.1
        with:
          # Url to Power Automate Flow to notify about Solution Deployment
          url: "${{ secrets.FLOW_HTTP_URL }}"
          # Request body to be sent the Flow with HTTP Trigger
          data: >-
            {
                "solutionFile": "${{ steps.deploy-solution.outputs.solution }}",
                "tag": "${{ steps.deploy-solution.outputs.solutionVersion }}",
                "environmentUrl": "${{ github.event.inputs.targetEnvironmentUrl }}",
                "repoUrl": "https://github.com/${{ github.repository }}",
                "runId": "${{ github.run_id }}",
                "sha": "${{ github.sha }}"
            }
```

## Credits

1. Wael Hamze for [Xrm CI Framework](https://github.com/WaelHamze/xrm-ci-framework)
2. Microsoft for [PowerShellForGitHub](https://github.com/microsoft/PowerShellForGitHub)
