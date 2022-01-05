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

| Name                             | Description                                                                                     |
|:---------------------------------|:------------------------------------------------------------------------------------------------|
| AWS_ACCOUNT_ID                   | AWS account id                                                                                  |
| AWS_CREDENTIALS_SECURE_FILE_NAME | Name of the secure file to download that contains AWS credentials                               |
| AWS_DEFAULT_REGION               | AWS default region                                                                              |
| CODEDEPLOY_BUCKET_NAME           | Name of the bucket that appspec files are uploaded to fo CodeDeploy                             |
| ENVIRONMENT                      | Name of the environment                                                                         |
| TF_CLI_ARGS_INIT                 | Arguments for Terraform init command. Generally this will include backend configuration values. |
| TEST_RESULTS_BUCKET_NAME         | Name of the bucket that integration and load test results are uploaded to.                      |

### Example
```yaml
variables:
  - template: variables/production.ap-southeast-2.yml@templates
```

## AWS
### Templates
#### CodeDeploy
##### Deploy
This template will copy the appspec file to the specified bucket and then trigger the deployment. If the Azure DevOps run is cancelled or fails and the deployment has begun then the deployment will be stopped.

The [deploy](./aws/codedeploy/deploy.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block. 

This template does require AWS credentials to be set up, which can be achieved using the [configure](#configure) template.

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
This template will tag the specified image, authenticate docker with ECR and then push the image.

The [push](./aws/ecr/push.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

This template does require AWS credentials to be set up, which can be achieved using the [configure](#configure) template.

###### Parameters
| Name           | Description                             | Type   | Default                  |
|:---------------|:----------------------------------------|:-------|:-------------------------|
| awsAccountId   | AWS account ID to push image to         | string | `$(AWS_ACCOUNT_ID)`      |
| awsRegion      | AWS region to push image to             | string | `$(AWS_DEFAULT_REGION)`  |
| imageName      | Name of the image to push               | string |                          |
| repositoryName | Name of the repository to push image to | string |                          |
| tag            | Tag of the image to push                | string | `$(Build.BuildNumber)`   |

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
This template will download the specified credential file and set an the `AWS_SHARED_CREDENTIALS_FILE` environment variable to the path of the credential file. The `AWS_SHARED_CREDENTIALS_FILE` will be used by the AWS CLI. 

The [configure](./aws/iam/configure.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

To use this template you will need to create a [secure file](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/secure-files?view=azure-devops) containing [AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-where) using the default profile.

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
This template will download the specified test result file from the specified S3 bucket and publish it to Azure DevOps.

The [publish test results](./aws/s3/publish-test-results.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

To use this template you will need to create a [secure file](https://docs.microsoft.com/en-us/azure/devops/pipelines/library/secure-files?view=azure-devops) containing [AWS credentials](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html#cli-configure-files-where) using the default profile.

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

The [build template](./docker/build.yml) is a [step template](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) meaning it needs to be nested under a `steps:` block.

##### Parameters
| Name             | Description                                                  | Type   | Default                     |
|:-----------------|:-------------------------------------------------------------|:-------|:----------------------------|
| arguments        | Additional arguments for the build step                      |        | " "                         |
| dockerfileName   | Name of the dockerfile in the working directory              | string | Dockerfile                  |
| tag              | Value to tag image with                                      |        | `$(Build.BuildNumber)`      |
| target           | Name of target to build. Also used for the name of the image | string |                             |
| workingDirectory | Directory where the related files are located                | string | `$(Build.SourcesDirectory)` |

##### Example
```yaml
- template: ./docker/build.yml@templates
  parameters: 
    target: build
```

#### Test
To run a container that will execute tests and write them to the mapped volume

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
To download an artifact created by the plan template, install terraform and apply the described changes in the `.tfplan` file.

The [build](./docker/build.yml) template is a [step](https://docs.microsoft.com/en-us/azure/devops/pipelines/process/templates?view=azure-devops#step-reuse) template meaning it needs to be nested under a `steps:` block.

If you are going to use this to apply changes to infrastructure in AWS you will need to configure the credentials using the [configure](#configure) template. 

##### Parameters
| Name             | Description                                                  | Type   | Default                     |
|:-----------------|:-------------------------------------------------------------|:-------|:----------------------------|
| arguments        | Additional arguments for the build step                      |        | " "                         |
| dockerfileName   | Name of the dockerfile in the working directory              | string | Dockerfile                  |
| tag              | Value to tag image with                                      |        | `$(Build.BuildNumber)`      |
| target           | Name of target to build. Also used for the name of the image | string |                             |
| workingDirectory | Directory where the related files are located                | string | `$(Build.SourcesDirectory)` |

##### Example
```yaml
- template: ./docker/build.yml@templates
  parameters: 
    target: build
```

#### Destroy

#### Plan


## Working examples
