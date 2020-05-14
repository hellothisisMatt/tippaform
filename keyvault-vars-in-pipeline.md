# Using Keyvault Variables in a Terraform .tfvars file (Multi Staged Pipelines)

```
Note: This guide assumes you've already created a keyvault within Azure and it has secret variables in it. 

To access these secret variables you need a service principal account that is authorized within Azure DevOps that will access the keyvault.
```

## Setting Up the Variable Group Library

1. In Azure DevOps navigate to:  `Pipelines --> Library`
2. Setup your variable group name and description.
3. Tick `Link secrets from an Azure key vault as variables`
4. Select your Azure subscription, and the keyvault you want to use.
5. Under the `Variables` section, click `Add` and select the secrets you want to include.

## Configuring Your `pipeline.yaml` File

1. At the top of your pipeline file, add the following snippet to include your variable group.

    ```
    variables:
    - group: my-variable-group
    - group: my-variable-group-linked-keyvault
    ```

2. Add a `replacetokens` step that will replace placeholders in your `*.tfvars` file with variables specified in your variable group library.

    ```
    - task: replacetokens@3
        displayName: "Add variable values to config file"
        inputs:
            rootDirectory: '$(Build.SourcesDirectory)'
            targetFiles: '**/pipeline_vars.tfvars'
            encoding: 'auto'
            writeBOM: true
            escapeType: 'none'
            verbosity: 'detailed'
            actionOnMissing: 'fail'
            keepToken: false
            tokenPrefix: '__'
            tokenSuffix: '__'

    ```


```
Note: If you're using a normal variable library where the library is NOT connected to a keyvault you do not have to explicitely redefine them in the Job's variables section. You can simply skip the rest of this section.
```

3. Within your Job's section, explicitely define your keyvault's secret variable as a Job scoped variable. (You cannot use the keyvault's variable directly)

    ```
    stages:
    - stage: Plan
      jobs:
        - job: Plan
          displayName: Create Terraform Plan Artifact
          variables:
            # keyvault-var-1 is defined in the keyvault and is linked to my-variable-group-linked-keyvault
            my-secret-var: $(keyvault-var-1)
            my-secret-var-2: $(keyvault-var-2)
          steps:
            ...

## Configuring Your `*.tfvars` File

1. For each Terraform variable create a placeholder value that follows this pattern:
    ```
    __my-var-name__
    ```
    This will make the `replacetokens` task look for a variable named `my-var-name` that is either in a variable group library (that isn't linked to a keyvault) imported at the top of `pipeline.yaml` config OR a variable defined in the Job's variables section.

2. Doing this with your file may look something like this:
    ```
    ...

    # Using my-variable-group's variables
    normal_var1 = "__normal-var-1__"
    normal_var2 = "__normal-var-2__"

    # Using job scoped variables from the keyvault
    var1 = "__my-secret-var__"
    var2 = "__my-secret-var-2__"
    
    ...
    ```

## Passing your `*.tfvars` file to your `terraform plan` step.

1. In your `pipeline.yaml` file visit your `terraform plan` step and add the following:

    ```
    - task: TerraformTaskV1@0
        displayName: "Create Terraform plan"
        inputs:
            provider: 'azurerm'
            command: 'plan'
            workingDirectory: '$(Build.SourcesDirectory)'
            # This is what you want to add!!!!!!!
            commandOptions: '-var-file pipeline/pipeline_vars.tfvars  -out tfplan'
    ```

    `-var-file path/to/pipeline_vars.tfvars` is the magic line.

## Done! Your `*.tfvars` file will be passed to your plan step with your variables added to it. 