# Azure Container Apps Build and Deploy

![](https://i.imgur.com/waxVImv.png)
### [View all Roadmaps](https://github.com/nholuongut/all-roadmaps) &nbsp;&middot;&nbsp; [Best Practices](https://github.com/nholuongut/all-roadmaps/blob/main/public/best-practices/) &nbsp;&middot;&nbsp; [Questions](https://www.linkedin.com/in/nholuong/)
<br/>

This action allows users to easily deploy their application source to an
[Azure Container App](https://azure.microsoft.com/en-us/services/container-apps/) in their GitHub workflow by either
providing a previously built image, a Dockerfile that an image can be built from, or using a builder to create a
runnable application image for the user.

If no Dockerfile is found or provided in the provided application source, the following steps are performed by this action:

- Uses the Oryx++ Builder to build the application source using [Oryx](https://github.com/microsoft/Oryx) to produce a
  runnable application image
- Pushes this runnable application image to the provided Azure Container Registry
- Creates or updates a Container App based on this image

If a Dockerfile is found or discovered in the application source, the builder won't be used and the image will be built
with a call to `docker build` and the Container App will be created or updated based on this image.

If a previously built image has already been pushed to the ACR instance and is provided to this action, no application
source is required and the image will be used when creating or updating the Container App.

## Prerequisites

Prior to running this action, a set of Azure resources and GitHub Actions are either required or optional depending on
the arguments provided to this action.

### Azure Container Registry

An [Azure Container Registry](https://azure.microsoft.com/en-us/products/container-registry/) must exist that the user
is able to push container images to. This action will leverage the Azure Container Registry to either push a built
runnable application image to and/or deploy a Container App from.

### Azure Container App environment

An [Azure Container App environment](https://docs.microsoft.com/en-us/azure/container-apps/environment) is recommended
to have been previously created by the user to improve the performance of the action. If no environment has been
created before, or if an environment cannot be found in the resource group that is being used to host the created
Container App, then an environment will be created by as a part of the `az containerapp up` command, which may take
additional time.

### `azure/login`

The [`azure/login`](https://github.com/marketplace/actions/azure-login) action is used to authenticate calls using the
Azure CLI, which is used in this action to call the
[`az containerapp up`](https://docs.microsoft.com/en-us/cli/azure/containerapp?view=azure-cli-latest#az-containerapp-up)
command. If `azure/login` is not called prior to this action being called in the GitHub workflow, the `azureCredentials`
argument can be used to provide the credentials needed to successfully call `azure/login`. These credentials are
recommended to be stored as a [GitHub secret](https://docs.github.com/en/actions/security-guides/encrypted-secrets),
and can be provided to this action without their value being exposed.

The credentials used for the `azure/login` action must have Contributor access over one of the following sets of
permissions:

- The existing Container App if both the `resourceGroup` and `containerAppName` arguments are provided _and_ exist
- The existing resource group if only the `resourceGroup` argument is provided _and_ exists
- The user's subscription if the `resourceGroup` argument is not provided _or_ is provided but does not exist

More information about configuring the deployment credentials required for this GitHub Action can be found
[here](https://github.com/marketplace/actions/azure-login#configure-deployment-credentials).

### `docker/login-action`

The [`docker/login-action`](https://github.com/marketplace/actions/docker-login) action is used to authenticate calls
to the user's Azure Container Registry, which will host the image that is then deployed to the Container App.
Currently, `docker/login-action` is called during every invocation of this action, so the user's Azure Container
Registry name is required, along with username and password credentials that are able to authenticate calls to this
Azure Container Registry. These credentials are able to be retrieved by
[creating a service principal](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-service-principal#create-a-service-principal)
and giving it proper permissions to the ACR resource.


## Arguments

Below are the arguments that can be provided to the Azure Container Apps Build and Deploy GitHub Action:

| Argument name             | Required | Description |
| ------------------------- | -------- | ----------- |
| `appSourcePath`           | No       | Absolute path on the GitHub runner of the source application code to be built. |
| `acrName`                 | Yes      | The name of the Azure Container Registry that the runnable application image will be pushed to. |
| `acrUsername`             | No       | The username used to authenticate push requests to the provided Azure Container Registry. If not provided, an access token will be generated via "az acr login" and provided to "docker login" to authenticate the requests. |
| `acrPassword`             | No       | The password used to authenticate push requests to the provided Azure Container Registry. If not provided, an access token will be generated via "az acr login" and provided to "docker login" to authenticate the requests. |
| `azureCredentials`        | No       | Azure credentials used by the `azure/login` action to authenticate Azure CLI requests if the user has not previously authenticated in the workflow calling this action. |
| `imageToBuild`            | No       | The custom name of the image that is to be built, pushed to ACR and deployed to the Container App by this action. _Note_: this image name should include the ACR server; _e.g._, `<acr-name>.azurecr.io/<repo>:<tag>`. If this argument is not provided, a default image name will be constructed in the form `<acr-name>.azurecr.io/github-action/container-app:<github-run-id>.<github-run-attempt>` |
| `imageToDeploy`           | No       | The custom name of the image that has already been pushed to ACR and will be deployed to the Container App by this action. _Note_: this image name should include the ACR server; _e.g._, `<acr-name>.azurecr.io/<repo>:<tag>`. If this argument is not provided, the value provided (or determined) for the `imageToBuild` argument will be used. |
| `dockerfilePath`          | No       | Relative path (_without file prefixes, see example below_) to the Dockerfile in the provided application source that should be used to build the image that is then pushed to ACR and deployed to the Container App. If not provided, this action will check if there is a file named `Dockerfile` in the provided application source and use that to build the image. Otherwise, the Oryx++ Builder will be used to create the image. |
| `containerAppName`        | No       | The name of the Container App that will be created or updated. If not provided, this value will be `github-action-container-app-<github-run-id>-<github-run-attempt>`. |
| `resourceGroup`           | No       | The resource group that the Container App will be created in, or currently exists in. If not provided, this value will be `<container-app-name>-rg`. |
| `containerAppEnvironment` | No       | The name of the Container App environment to use with the application. If not provided, an existing environment in the resource group of the Container App will be used, otherwise, an environment will be created in the formation `<container-app-name>-env`. |
| `runtimeStack`            | No       | The platform version stack used in the final runnable application image that is deployed to the Container App. The value should be provided in the formation `<platform>:<version>`. If not provided, this value is determined by Oryx based on the contents of the provided application. Please refer to [this document](https://github.com/microsoft/Oryx/blob/main/doc/supportedRuntimeVersions.md) for more information on supported runtime stacks for Oryx. |
| `targetPort`              | No       | The target port that the Container App will listen on. If not provided, this value will be "80" for Python applications and "8080" for all other supported platforms. |

## Usage

See [`action.yml`](./action.yml)

### Minimal

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `<container-app-name>-rg`.

### Container App name provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      containerAppName: my-test-container-app
```

This will create a new Container App named `my-test-container-app` in a new resource group named
`my-test-container-app-rg`.

### Resource group provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      resourceGroup: my-test-rg
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `my-test-rg`.

### Container App name and resource group provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      containerAppName: my-test-container-app
      resourceGroup: my-test-rg
```

If the `my-test-rg` resource group does not exist, this will create the resource group and create a new Container App
named `my-test-container-app` within the resource group. If the resource group already exists, this will create a new
Container App named `my-test-container-app` in the resource group, or update the Container App if it already exists
within the resource group.

### Container App environment provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      containerAppEnvironment: my-test-container-app-env
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `<container-app-name>-rg` with a new Container App environment named `my-test-container-app-env`.

### Runtime stack provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      runtimeStack: 'dotnetcore:7.0'
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `<container-app-name>-rg` where the runnable application image is using the .NET 7 runtime stack.

### Dockerfile provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      dockerfilePath: test.Dockerfile
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `<container-app-name>-rg` where the runnable application image was created from the `test.Dockerfile`
file found in the provided application source path directory.

_Note_: for values provided to `dockerfilePath`, no file prefixes should be included (_e.g._, `./test.Dockerfile` should be
passed as just `test.Dockerfile`). The provided `appSourcePath` and `dockerfilePath` arguments will be concatenated inside
of the GitHub Action.

### Image to build provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      imageToBuild: mytestacr.azurecr.io/app:latest
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `<container-app-name>-rg` where the image built and pushed to ACR is named
`mytestacr.azurecr.io/app:latest`

### Image to deploy provided

```yml
steps:

  - name: Log in to Azure
    uses: azure/login@v1
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}

  - name: Build and deploy Container App
    uses azure/container-app-deploy-action@v0
    with:
      appSourcePath: ${{ github.workspace }}
      acrName: mytestacr
      acrUsername: ${{ secrets.REGISTRY_USERNAME }}
      acrPassword: ${{ secrets.REGISTRY_PASSWORD }}
      imageToDeploy: mytestacr.azurecr.io/app:latest
```

This will create a new Container App named `github-action-container-app-<github-run-id>-<github-run-attempt>` in a new
resource group named `<container-app-name>-rg` where **no new image is built**, but an existing image in ACR named
`mytestacr.azurecr.io/app:latest` will be deployed to the Container App.

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or

# 🚀 I'm are always open to your feedback.  Please contact as bellow information:
### [Contact ]
* [Name: nho Luong]
* [Skype](luongutnho_skype)
* [Github](https://github.com/nholuongut/)
* [Linkedin](https://www.linkedin.com/in/nholuong/)
* [Email Address](luongutnho@hotmail.com)

![](https://i.imgur.com/waxVImv.png)
![](Donate.png)
[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/nholuong)

# License
* Nho Luong (c). All Rights Reserved.🌟

