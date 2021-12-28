# Azure DevOps for Expensely pipelines

This repository contains task and variable templates for Azure DevOps.

To use these templates, add a resource section to the top of the pipeline file.  
[Read more about resources](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops&tabs=schema)  
Task template example:
```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: expensely/azure-devops-templates
      endpoint: expensely
```

Variable template example:
```yaml
variables:
  - template: variables/production.ap-southeast-2.yml@expensely-templates
```

## Docker
#### Build
To build the specified target in a `Dockerfile`.

The [build template](./docker/build.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

The `target` parameter will be used as the name.

##### Parameters
| Name           | Description             | Type   | Default                |
|:---------------|:------------------------|:-------|:-----------------------|
| dockerfilePath | Path of the dockerfile  | string | Dockerfile             |
| target         | Name of target to build | string |                        |
| tag            | Tag of image            | string | `$(Build.BuildNumber)` |

##### Example
```yaml
- template: ./docker/build.yml@templates
  parameters: 
    dockerfilePath: src/Dockerfile
    target: build
    tag: 1.0.0-preview
```

#### Test
To run a container that will execute tests

The [test template](./docker/test.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

The `target` parameter will be used as the name.

##### Parameters
| Name                          | Description                                                | Type   | Default                             |
|:------------------------------|:-----------------------------------------------------------|:-------|:------------------------------------|
| containerTestResultsDirectory | Path in the container to where test results are written to | string | `/artifacts/tests`                  |
| containerName                 | Name of the container to run                               | string | tests                               |
| coverageThreshold             | Threshold for code coverage                                | string | 60                                  |
| pathToSources                 | Directory containing source code                           | string | `$(System.DefaultWorkingDirectory)` |
| tag                           | Tag of image                                               | string | `$(Build.BuildNumber)`              |
| testResultsFormat             | Format of test result/s file                               | string | VSTest                              |

##### Example
```yaml
- template: ./docker/test.yml@templates
  parameters:
    containerTestResultsDirectory: /artifacts/tests
    containerName: tests
    tag: $(Build.BuildNumber)
    pathToSources: $(System.DefaultWorkingDirectory)
    testResultsFormat: VSTest
    coverageThreshold: 60
```
