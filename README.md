# Azure DevOps for Expensely pipelines

Reference repository
```yaml
resources:
  repositories:
    - repository: expensely-templates
      type: github
      name: expensely/azure-devops-templates
      endpoint: expensely
```

Consume variables
```yaml
variables:
  - template: variables/production.ap-southeast-2.yml@expensely-templates
```