# Azure DevOps templates for Expensely

This repository contains task and variable templates for Azure DevOps.

To use these templates, add a [resource](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/resources?view=azure-devops&tabs=schema) section to the top of the pipeline file.  
Task template example:
```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: expensely/azure-devops-templates
      endpoint: expensely
```

## Variables
Non-secret variables will be automatically added as environment variables which can be consumed without any mapping. The variables listed below are the minimum needed to use all templates.

| Name                               | Description                                                                                     |
|:-----------------------------------|:------------------------------------------------------------------------------------------------|
| `AWS_ACCOUNT_ID`                   | AWS account id                                                                                  |
| `AWS_CREDENTIALS_SECURE_FILE_NAME` | Name of the secure file to download that contains AWS credentials                               |
| `AWS_DEFAULT_REGION`               | AWS default region                                                                              |
| `CODEDEPLOY_BUCKET_NAME`           | Name of the bucket that appspec files are uploaded to fo CodeDeploy                             |
| `ENVIRONMENT`                      | Name of the environment                                                                         |
| `TF_CLI_ARGS_INIT`                 | Arguments for Terraform init command. Generally this will include backend configuration values. |
| `TEST_RESULTS_BUCKET_NAME`         | Name of the bucket that integration and load test results are uploaded to.                      |

### Example
```yaml
variables:
  - template: variables/production.ap-southeast-2.yml@templates
```

## AWS
### Templates
#### CodeDeploy
##### Deploy
Trigger a CodeDeploy deployment.

This template will:
1. Push the AppSpec file to the specified path in the S3 bucket
2. Trigger the deployment
3. Cancel the deployment if the run is cancelled or has failed and the deployment has begun

The [deploy](./aws/codedeploy/deploy.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block. 

This template does require AWS credentials to be set up, which can be achieved using the [configure](#configure) template.

###### Required environment variables
* `CODEDEPLOY_BUCKET_NAME`

###### Parameters
| Name             | Description                                             | Type   | Default                 |
|:-----------------|:--------------------------------------------------------|:-------|:------------------------|
| destinationPath  | Path(key) in the bucket to upload the specified file to | string |                         |
| appSpecFileName  | Name of the appspec file to upload                      | string |                         |
| workingDirectory | Directory where the app spec file is located            | string | `$(Pipeline.Workspace)` |

###### Example
```yaml
steps:
  - template: ./aws/codedeploy/deploy.yml@templates
    parameters:
      destinationPath: time/preview23
      fileName: 1.0.1234.1.yml
```

#### ECR
##### Push
This template will push a Docker image to ECR.

This template will:
1. Tag the image with relevant ECR url
2. Login Docker to ECR 
3. Push the tagged image

The [push](./aws/ecr/push.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

This template does require AWS credentials to be set up, which can be achieved using the [configure](#configure) template.

###### Required environment variables
* `AWS_ACCOUNT_ID`
* `AWS_DEFAULT_REGION`

###### Parameters
| Name           | Description                             | Type   | Default                 |
|:---------------|:----------------------------------------|:-------|:------------------------|
| awsAccountId   | AWS account ID to push image to         | string | `$(AWS_ACCOUNT_ID)`     |
| awsRegion      | AWS region to push image to             | string | `$(AWS_DEFAULT_REGION)` |
| imageName      | Name of the image to push               | string |                         |
| repositoryName | Name of the repository to push image to | string |                         |
| tag            | Tag of the image to push                | string | `$(Build.BuildNumber)`  |

###### Example
```yaml
steps:
  - template: ./aws/ecr/push.yml@templates
    parameters:
      imageName: migration
      repositoryName: migration
```

#### IAM
##### Configure
This template will configure the AWS credentials

This template will:
1. Download the secure file
2. Set the `AWS_SHARED_CREDENTIALS_FILE` environment variable with the path of the file

The [configure](./aws/iam/configure.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

To use this template you will need to create a [secure file](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/secure-files?view=azure-devops) containing [AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-where) using the default profile.

###### Required environment variables
* `AWS_CREDENTIALS_SECURE_FILE_NAME`

###### Parameters
| Name       | Description                                                                                                                                                                      | Type   | Default                               |
|:-----------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:-------|:--------------------------------------|
| secureFile | Name of the secure file to download. The default value is the value from the `AWS_CREDENTIALS_SECURE_FILE_NAME` variable set in the variable file mentioned [above](#variables). | string | `$(AWS_CREDENTIALS_SECURE_FILE_NAME)` |

###### Example
```yaml
steps:
  - template: ./aws/iam/configure.yml@templates
    parameters:
      secureFile: aws.production.ap-southeast-2
```

#### S3
##### Publish tests results
This template publish test results to Azure DevOps that are located in an S3 bucket.

This template will:
1. Copy the test results file from S3
2. Publish the test results file

The [publish test results](./aws/s3/publish-test-results.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

To use this template you will need to create a [secure file](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/secure-files?view=azure-devops) containing [AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-where) using the default profile.

###### Required environment variables
* `TEST_RESULTS_BUCKET_NAME`

###### Parameters
| Name                | Description                           | Type   | Default                 |
|:--------------------|:--------------------------------------|:-------|:------------------------|
| sourcePath          | Key of the test results file          | string |                         |
| testResultsFileName | Name of test results file             | string |                         |
| testResultsFormat   | Tests results format                  | string | `JUnit`                 |
| testRunTitle        | Title of the test run                 | string |                         |
| workingDirectory    | Directory where the files are located | string | `$(Pipeline.Workspace)` |

###### Example
```yaml
steps:
  - template: ./aws/s3/publish-test-results.yml@templates
    parameters:
      sourcePath: time/$(ENVIRONMENT)$(System.PullRequest.PullRequestNumber)
      fileName: $(NPM_BUILD_NUMBER).xml
      testRunTitle: Integration
```

## Azure DevOps
### Templates
#### Approve
This template will provide a manual validation step in the pipeline.

The [publish-test-results.yml](./azure-devops/approve.yml) is a [job template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#job-reuse) meaning it needs to be nested under a `jobs:` block.

###### Parameters
| Name                   | Description                                        | Type   | Default                    |
|:-----------------------|:---------------------------------------------------|:-------|:---------------------------|
| dependsOn              | Name of job that needs to complete before this job | string |                            |
| timeoutInMinutes       | Number of minutes until this job times out         | number | 60                         |
| userToNotify           | Users to notify                                    | string |                            |
| validationInstructions | Validation instructions                            | string | Validate the dependant job |

###### Example
```yaml
jobs:
  - template: job/approve.yml@templates
    parameters:
      dependsOn: plan
      timeoutInMinutes: 60
      notifyUsers: '[Expensely]\Expensely Team'
```

## Docker
### Templates
#### Build
To build the specified target in a `Dockerfile` and tag the image.

This template will:
1. Build the target and tag the image

The [build template](./docker/build.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

##### Parameters
| Name             | Description                                                  | Type   | Default                     |
|:-----------------|:-------------------------------------------------------------|:-------|:----------------------------|
| arguments        | Additional arguments for the build step                      | string | " "                         |
| dockerfileName   | Name of the dockerfile in the working directory              | string | Dockerfile                  |
| tag              | Value to tag image with                                      | string | `$(Build.BuildNumber)`      |
| target           | Name of target to build. Also used for the name of the image | string |                             |
| workingDirectory | Directory where the related files are located                | string | `$(Build.SourcesDirectory)` |

##### Example
```yaml
- template: ./docker/build.yml@templates
  parameters: 
    target: build
```

#### Test
To run a container that will execute tests and write them to the mapped volume.

This template will:
1. Map a volume to the tests result folder and run the container
2. Merge the Cobertura reports. If you this doesn't happen then the code coverage isn't accurate as only 1 report is used for reporting
3. Publish the merged code coverage files
4. Publish the test results
5. Validate code coverage against the coverage threshold

The [test template](./docker/test.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

##### Parameters
| Name                          | Description                                                | Type   | Default                             |
|:------------------------------|:-----------------------------------------------------------|:-------|:------------------------------------|
| containerName                 | Name of the container to run                               | string | tests                               |
| containerTestResultsDirectory | Path in the container to where test results are written to | string | `/artifacts/tests`                  |
| coverageThreshold             | Threshold for code coverage                                | string | 60                                  |
| pathToSources                 | Directory containing source code                           | string | `$(System.DefaultWorkingDirectory)` |
| tag                           | Tag of image                                               | string | `$(Build.BuildNumber)`              |
| testResultsFormat             | Format of test result/s file                               | string | VSTest                              |

##### Example
```yaml
- template: ./docker/test.yml@templates
```

## Terraform
### Templates
#### Apply 
Apply the changes described in the `.tfplan` file

This template will:
1. Download the specified artifact
2. Extract the files from the downloaded artifact
3. Install the specified Terraform version
4. Apply the Terraform changes

The [apply](./terraform/apply.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

If you are going to use this to apply changes to infrastructure in AWS you will need to configure the credentials using the [configure](#configure) template. 

##### Parameters
| Name                          | Description                                    | Type   | Default                 |
|:------------------------------|:-----------------------------------------------|:-------|:------------------------|
| applyAdditionalCommandOptions | Additional command options for Terraform apply | string | " "                     |
| artifactName                  | Name of the published artifact to download     | string |                         |
| version                       | Terraform version to download and install      | string | 1.0.2                   |
| workingDirectory              | Working directory                              | string | `$(Pipeline.Workspace)` |

##### Example
```yaml
- template: ./terraform/apply.yml@templates
  parameters:
    artifactName: preview.terraform.plan
```

#### Destroy
Destroy infrastructure and delete the relevant workspace

This template will:
1. Install the specified Terraform version
2. Initialise the Terraform
3. Select the relevant workspace
4. Destroy the infrastructure
5. Delete the workspace

##### Required environment variables
* `TF_CLI_ARGS_INIT`

##### Parameters
| Name                         | Description                                              | Type   | Default                                    |
|:-----------------------------|:---------------------------------------------------------|:-------|:-------------------------------------------|
| destroyAdditionalArguments   | Additional command options for Terraform destroy command | string | " "                                        |
| initAdditionalCommandOptions | Additional command options for Terraform init command    | string | " "                                        |
| version                      | Terraform version to download and install                | string | 1.0.2                                      |
| workingDirectory             | Directory where Terraform files are located              | string | `$(Build.SourcesDirectory)/infrastructure` |
| workspaceName                | Terraform workspace                                      | string |                                            |

##### Example
```yaml
- template: ./terraform/destroy.yml@templates
  parameters:
    workspaceName: time-preview-23
```

#### Plan
Create a Terraform plan.

This template will:
1. Install the specified Terraform version
2. Initialise the Terraform
3. Select or create the relevant workspace
4. Plan the changes and save them to a file
5. Archive the changes to tar.gz 
6. Publish the archive

This template will install the specified Terraform version, initialise Terraform,   

and apply the changes described in `.tfplan` file.

The [apply](./terraform/apply.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

If you are going to use this to apply changes to infrastructure in AWS you will need to configure the credentials using the [configure](#configure) template.

##### Required environment variables
* `TF_CLI_ARGS_INIT`

##### Parameters
| Name                         | Description                                                                          | Type   | Default                                    |
|:-----------------------------|:-------------------------------------------------------------------------------------|:-------|:-------------------------------------------|
| artifactName                 | Name of the published artifact, that contains the plan file, to download and extract | string |                                            |
| initAdditionalCommandOptions | Additional command options for Terraform init                                        | string | " "                                        |
| planAdditionalCommandOptions | Additional command options for the Terraform plan                                    | string | " "                                        |
| version                      | Terraform version to download and install                                            | string | 1.0.2                                      |
| workingDirectory             | Directory where Terraform files are located                                          | string | `$(Build.SourcesDirectory)/infrastructure` |
| workspaceName                | Terraform workspace                                                                  | string |                                            |

##### Example
```yaml
- template: ./terraform/plan.yml@templates
  parameters:
    artifactName: preview.terraform.plan
    workspaceName: time-preview-23
```

## Working example
### Multi-Stage
```yaml
resources:
  repositories:
    - repository: templates
      type: github
      name: expensely/azure-devops-templates
      endpoint: expensely

trigger: none

pr:
  branches:
    include:
      - 'main'

pool:
  vmImage: ubuntu-latest

variables:
  - template: variables/preview.ap-southeast-2.yml@templates

stages:
  - stage: build
    displayName: Build
    jobs:
      - job: setup
        displayName: Setup
        steps:
          - checkout: none
          - script: echo "##vso[build.updatebuildnumber]1.0.$(build.buildid).$(System.StageAttempt)"
            displayName: Set build identifier
          - script: echo "##vso[task.setvariable variable=NPMBuildNumber;isOutput=true]1.0.$(build.buildid)-$(System.StageAttempt)"
            displayName: Set NPM build identifier
            name: setNPMBuildIdentifier

      - job: api
        displayName: Api
        dependsOn:
          - setup
        steps:
          - template: ./docker/build.yml@templates
            parameters:
              target: base
              arguments: "--build-arg BUILD_NUMBER=$(Build.BuildNumber) --build-arg PAT=$(System.AccessToken)"
          - template: ./docker/build.yml@templates
            parameters:
              target: test
              arguments: "--build-arg BUILD_NUMBER=$(Build.BuildNumber) --build-arg PAT=$(System.AccessToken)"
          - template: ./docker/test.yml@templates
            parameters:
              containerName: test
              coverageThreshold: 15
          - template: ./docker/build.yml@templates
            parameters:
              target: api
              arguments: "--build-arg BUILD_NUMBER=$(Build.BuildNumber) --build-arg PAT=$(System.AccessToken)"
          - template: ./docker/build.yml@templates
            parameters:
              target: migration
              arguments: "--build-arg BUILD_NUMBER=$(Build.BuildNumber) --build-arg PAT=$(System.AccessToken)"

          - template: aws/iam/configure.yml@templates

          - template: ./aws/ecr/push.yml@templates
            parameters:
              imageName: api
              repositoryName: time-api
          - template: ./aws/ecr/push.yml@templates
            parameters:
              imageName: migration
              repositoryName: time-migration

      - job: integration_tests
        displayName: Integration Tests
        dependsOn:
          - setup
        variables:
          NPM_BUILD_NUMBER: $[ dependencies.setup.outputs['setNPMBuildIdentifier.NPMBuildNumber'] ]
        steps:
          - template: ./docker/build.yml@templates
            parameters:
              target: integration-tests
              workingDirectory: tests/Time.IntegrationTests
              arguments: "--build-arg BUILD_NUMBER=$(NPM_BUILD_NUMBER)"
              tag: $(NPM_BUILD_NUMBER)

          - template: aws/iam/configure.yml@templates

          - template: ./aws/ecr/push.yml@templates
            parameters:
              imageName: integration-tests
              repositoryName: time-integration-tests
              tag: $(NPM_BUILD_NUMBER)

  - stage: preview
    displayName: Preview
    dependsOn:
      - build
    variables:
      - name: ARTIFACT_NAME
        value: preview.terraform
        readonly: true
      - name: NPM_BUILD_NUMBER
        value: $[ stageDependencies.build.setup.outputs['setNPMBuildIdentifier.NPMBuildNumber'] ]
    jobs:
      - job: plan
        displayName: Plan
        steps:
          - template: aws/iam/configure.yml@templates
          - template: terraform/plan.yml@templates
            parameters:
              artifactName: $(ARTIFACT_NAME)
              workspaceName: service-time-$(ENVIRONMENT)$(System.PullRequest.PullRequestNumber)
              planAdditionalCommandOptions: '-var-file="variables/$(ENVIRONMENT).$(AWS_DEFAULT_REGION).tfvars" -var="build_identifier=$(Build.BuildNumber)" -var="environment=Preview$(System.PullRequest.PullRequestNumber)" -var="subdomain=time$(System.PullRequest.PullRequestNumber)" -var="npm_build_identifier=$(NPM_BUILD_NUMBER)"'
      - deployment: deploy
        displayName: Deploy
        dependsOn:
          - plan
        environment: preview
        strategy:
          runOnce:
            deploy:
              steps:
                - download: none
                - template: aws/iam/configure.yml@templates
                - template: terraform/apply.yml@templates
                  parameters:
                    artifactName: $(ARTIFACT_NAME)
                - template: aws/codedeploy/deploy.yml@templates
                  parameters:
                    fileName: $(Build.BuildNumber).yaml
                    destinationPath: time/$(ENVIRONMENT)$(System.PullRequest.PullRequestNumber)
                - template: aws/s3/publish-test-results.yml@templates
                  parameters:
                    sourcePath: time/$(ENVIRONMENT)$(System.PullRequest.PullRequestNumber)
                    fileName: $(NPM_BUILD_NUMBER).xml
                    testRunTitle: Integration
```


