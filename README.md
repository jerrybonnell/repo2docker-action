![](https://github.com/jupyterhub/repo2docker-action/workflows/Test/badge.svg) [![MLOps](https://img.shields.io/badge/MLOps-black.svg?logo=github&?logoColor=blue)](https://mlops-github.com)


# <a href="https://github.com/jupyterhub/repo2docker"><img src="https://raw.githubusercontent.com/jupyterhub/repo2docker/71eb8058c790a88d223470a55f3ea5b744614dcf/docs/source/_static/images/repo2docker.png" height="40px" /></a>  repo2docker GitHub Action


<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [What Can I Do With This Action?](#what-can-i-do-with-this-action)
- [API Reference](#api-reference)
	- [Mandatory Inputs](#mandatory-inputs)
	- [Optional Inputs](#optional-inputs)
	- [Outputs](#outputs)
- [Testing the built image](#testing-the-built-image)
- [Examples](#examples)
	- [mybinder.org](#mybinderorg)
		- [Cache builds on mybinder.org](#cache-builds-on-mybinderorg)
		- [Cache Builds On mybinder.org And Provide A Link](#cache-builds-on-mybinderorg-and-provide-a-link)
		- [Use GitHub Actions To Cache The Build For BinderHub](#use-github-actions-to-cache-the-build-for-binderhub)
	- [Push Repo2Docker Image To DockerHub](#push-repo2docker-image-to-dockerhub)
	- [Push Repo2Docker Image To quay.io](#push-repo2docker-image-to-quayio)
	- [Push Repo2Docker Image To Amazon ECR](#push-repo2docker-image-to-amazon-ecr)
	- [Push Repo2Docker Image To Google Container Registry](#push-repo2docker-image-to-google-container-registry)
	- [Push Repo2Docker Image To Google Artifact Registry](#push-repo2docker-image-to-google-artifact-registry)
	- [Push Repo2Docker Image To Azure Container Registry](#push-repo2docker-image-to-azure-container-registry)
	- [Push Repo2Docker Image To Other Registries](#push-repo2docker-image-to-other-registries)
	- [Change Image Name](#change-image-name)
	- [Test Image Build](#test-image-build)
- [Contributing](#contributing-to-repo2docker-action)

<!-- /TOC -->


Trigger [repo2docker](https://github.com/jupyter/repo2docker) to build a Jupyter enabled Docker image from your GitHub repository and push this image to a Docker registry of your choice.  This will automatically attempt to build an environment from configuration files found in your repository in the [manner described here](https://repo2docker.readthedocs.io/en/latest/usage.html#where-to-put-configuration-files).

Read the full docs on repo2docker for more information:  https://repo2docker.readthedocs.io

Images generated by this action are automatically tagged with both `latest` and `<SHA>` corresponding to the relevant [commit SHA on GitHub](https://help.github.com/en/github/getting-started-with-github/github-glossary#commit).  Both tags are pushed to the Docker registry specified by the user. If an existing image with the `latest` tag already exists in your registry, this Action attempts to pull that image as a cache to reduce uncessary build steps.

# What Can I Do With This Action?

- Use repo2docker to pre-cache images for your own [BinderHub cluster](https://binderhub.readthedocs.io/en/latest/zero-to-binderhub/setup-binderhub.html), or for [mybinder.org](https://mybinder.org/).
  - You can use this Action to pre-cache Docker images to a Docker registry that you can reference in your repo.  For example if you have the file `Dockerfile` in the `binder/` directory relative to the root of your repository with the following contents, this will allow Binder to start quickly by pulling an image you have already built:

    ```dockerfile
    # This is the image that is built and pushed by this Action (replace this with your image name)
    FROM myorg/myimage:latest
    ...
    ```
- Provide a way to Dockerize data science repositories with Jupyter server enabled that you can deploy to VMs, [serverless computing](https://en.wikipedia.org/wiki/Serverless_computing) or other services that can serve Docker containers as-a-service.
- Maximize reproducibility by allowing authors, without any prior knowledge of Docker, to build and share containers.
- Run tests after the image has been built, to make sure package changes don't break your code.

# API Reference

See the [examples](#examples) section is very helpful for understanding the inputs and outputs of this Action.

## Optional Inputs

- **`DOCKER_USERNAME`**:
    description: Docker registry username.  If not supplied, credentials must be setup ahead of time.
- **`DOCKER_PASSWORD`**:
    description: Docker registry password or [access token (recommended)](https://docs.docker.com/docker-hub/access-tokens/).  If not supplied, credentials must be setup ahead of time.
- **`DOCKER_REGISTRY`**:
    description: domain name of the docker registry.  If not supplied, this defaults to [DockerHub](https://hub.docker.com/)
- **`IMAGE_NAME`**:
    name of the image.  Example - myusername/myContainer.  If not supplied, this defaults to `<DOCKER_USERNAME>/<GITHUB_REPOSITORY_NAME>` or `<GITHUB_ACTOR>/<GITHUB_REPOSITORY_NAME>`.
- **`NOTEBOOK_USER`**:
    description: username of the primary user in the image. If this is not specified, this is set to `joyvan`.  **NOTE**: This value is also overriden with `jovyan` if the parameters `BINDER_CACHE` or `MYBINDERORG_TAG` are provided.
- **`REPO_DIR`**:
    Path inside the image where contents of the repositories are copied to, and where all the build operations (such as postBuild) happen. Defaults to `/home/<NOTEBOOK_USER>` if not set.
- **`APPENDIX_FILE`**:
    Path to file containing Dockerfile commands to run at the end of the build. Can be used to customize the resulting image after all standard build steps finish.
- **`LATEST_TAG_OFF`**:
    Setting this variable to any value will prevent your image from being tagged with `latest`. Note that your image is always tagged with the [GitHub commit SHA](https://help.github.com/en/github/getting-started-with-github/github-glossary#commit).  
- **`ADDITIONAL_TAG`**:
    An optional string that specifies the name of an additional tag you would like to apply to the image.  Images are already tagged with the relevant [GitHub commit SHA](https://help.github.com/en/github/getting-started-with-github/github-glossary#commit).
- **`NO_PUSH`**:
    Setting this variable to any value will prevent any images from being pushed to a registry.  Furthermore, verbose logging will be enabled in this mode.  This is disabled by default.
- **`BINDER_CACHE`**:
    Setting this variable to any value will add the file `binder/Dockerfile` that references the docker image that was pushed to the registry by this Action.  You cannot use this option if the parameter `NO_PUSH` is set.  This is disabled by default.
    - Note: This Action assumes you are not explicitly using Binder to build your dependencies (You are using this Action to build your dependencies).  If a directory `binder` with other files other than `Dockerfile` or a directory named `.binder/` is detected, this step will be aborted.  This Action does not support caching images for Binder where dependencies are defined in `binder/Dockerfile` (if you are defining your dependencies this way, you probably don't need this Action).

      When this parameter is supplied, this Action will add/override `binder/Dockerfile` in the branch checked out in the Actions runner:
      ```dockerfile
      ### DO NOT EDIT THIS FILE! This Is Automatically Generated And Will Be Overwritten ###
      FROM <IMAGE_NAME>
      ```
- **`COMMIT_MSG`**:
     The commit message associated with specifying the `BINDER_CACHE` flag.  If no value is specified, the default commit message of `Update image tag` will be entered.
- **`MYBINDERORG_TAG`**:
    This the Git branch, tag, or commit that you want [mybinder.org](https://mybinder.org/) to proactively build from your repo.  This is useful if you wish to reduce startup time on mybinder.org.  **Your repository must be public for this work, as mybinder.org only works with public repositories**.
- **`PUBLIC_REGISTRY_CHECK`**:
    Setting this variable to any value will validate that the image pushed to the registry is publicly visible.

## Outputs

- **`IMAGE_SHA_NAME`**
    The name of the docker image, which is tagged with the SHA.
- **`PUSH_STATUS`**:
    This is `false` if `NO_PUSH` is provided or `true` otherwhise.

# Testing the built image

You can automatically test your built image to make sure package additions or
removals do not break your code, allowing you to make changes with confidence.
[pytest](https://docs.pytest.org/) is used to run the tests, and
[pytest-notebook](https://pytest-notebook.readthedocs.io/) is used to
run any Jupyter Notebooks as tests.

This **works with any Jupyter kernel**. This action will use the Jupyter kernel defined in any notebook you put in `image-tests/`. This can be used to execute and test notebooks from any language.

To use automatic image testing, follow these steps:

1. Create a directory named `image-tests/` in your GitHub repository.
2. Any `.py` files you add inside this directory will be discovered
   and run with `pytest` inside the built image after the image has
   successfully built.
3. Any Jupyter Notebook (`.ipynb`) files inside this directory will
   be run with [`pytest-notebook`](https://pytest-notebook.readthedocs.io/en/latest/), and the notebook is considered to
   have *failed* if the outputs of the code execution do not match
   the outputs already in the notebook. A nice diff of the outputs
   is shown if they differ. See the [pytest-notebook docs](https://pytest-notebook.readthedocs.io/en/latest/)
   for more information.
4. Optionally, a `requirements.txt` file inside the `image-tests/`
   directory can list additional libraries installed just for the
   test.

For example, look at the following image environment repository structure:

```
my-image/
├── environment.yml
└── image-tests
    ├── mytestnotebook.ipynb
    └── mytest.py
```

This defines three things:

- **`environment.yml`** is a repo2docker environment file, which defines the packages for the user image
- **`image-tests/mytestnotebook.ipynb`** is a Jupyter notebook that is already executed so its outputs are included in the `ipynb` file. When the image is built, this notebook will be re-executed, and the outputs compared against the version stored with the repository.
- **`image-tests/mytest.py`** is a Python file that will be run with Pytest, and any failures will be reported.
# Examples

## mybinder.org

A very popular use case for this Action is to cache builds for [mybinder.org](https://mybinder.org/).  If you desire to cache builds for mybinder.org, you must specify the argument `MYBINDERORG_TAG`.  Some examples of doing this are below:

### Cache builds on mybinder.org

Proactively build your environment on mybinder.org for any branch.  Alternatively, you can use [using GitHub Actions to build an image for BindHub generally](https://github.com/jupyterhub/repo2docker-action#use-github-actions-to-cache-the-build-for-binderhub), including mybinder.org.

```yaml
name: Binder
on: [push]

jobs:
  Create-MyBinderOrg-Cache:
    runs-on: ubuntu-latest
    steps:
    - name: cache binder build on mybinder.org
      uses: jupyterhub/repo2docker-action@master
      with:
        NO_PUSH: true
        MYBINDERORG_TAG: ${{ github.event.ref }} # This builds the container on mybinder.org with the branch that was pushed on.
```

### Cache Builds On mybinder.org And Provide A Link

Same example as above, but also comment on a PR with a link to the binder environment.  Commenting on the PR is optional, and is included here for informational purposes only.  In this example the image will only be cached when the pull request is opened but not if the pull request is updated with subsequent commits.

In this example the image will only be cached when the pull request is opened but not if the pull request is updated with subsequent commits.

```yaml
name: Binder
on:
  pull_request:
    types: [opened, reopened]

jobs:
  Create-Binder-Badge:
    runs-on: ubuntu-latest
    steps:
    - name: cache binder build on mybinder.org
      uses: jupyterhub/repo2docker-action@master
      with:
        NO_PUSH: true
        MYBINDERORG_TAG: ${{ github.event.pull_request.head.ref }}

    - name: comment on PR with Binder link
      uses: actions/github-script@v1
      with:
        github-token: ${{secrets.GITHUB_TOKEN}}
        script: |
          var BRANCH_NAME = process.env.BRANCH_NAME;
          github.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: `[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/${context.repo.owner}/${context.repo.repo}/${BRANCH_NAME}) :point_left: Launch a binder notebook on this branch`
          })
      env:
        BRANCH_NAME: ${{ github.event.pull_request.head.ref }}
```        

### Use GitHub Actions To Cache The Build For BinderHub

Instead of forcing mybinder.org to cache your builds, you can optionally build a Docker image with GitHub Actions and push that to a Docker registry, so that any [BinderHub](https://binderhub.readthedocs.io/en/latest/) instance, including mybinder.org only has to pull the image.  This might give you more control than triggering a build directly on mybinder.org like the method illustrated above.  In this example, you must supply the [secrets](https://help.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets) `DOCKER_USERNAME` and `DOCKER_PASSWORD` so that Actions can push to DockerHub.  Note that, instead of your actual password, you can use an [access token](https://docs.docker.com/docker-hub/access-tokens/) — which may be a more secure option.

In this case, we set `BINDER_CACHE` to `true` to enable this option.  See the documentation for the parameter `BINDER_CACHE` in the [Optional Inputs](#optional-inputs) section for more information.

```yaml
name: Test
on: push

jobs:
  binder:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v2
      with:
        ref: ${{ github.event.pull_request.head.sha }}

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        BINDER_CACHE: true
        PUBLIC_REGISTRY_CHECK: true
```

## Push Repo2Docker Image To DockerHub

We recommend creating a [personal access token](https://docs.docker.com/docker-hub/access-tokens/)
and use that as `DOCKER_PASSWORD` instead of using your dockerhub password.

```yaml
name: Build Notebook Container
on: [push] # You may want to trigger this Action on other things than a push.
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

## Push Repo2Docker Image To quay.io

DockerHub now has some [pretty strong rate limits](https://docs.docker.com/docker-hub/download-rate-limit/),
so you might want to push to a different docker repository. 
[quay.io](https://quay.io/) is a popular place, and isn't tied
to any particular cloud vendor.

1. Login to [quay.io](https://quay.io)
2. Create a new [repository](https://quay.io/new/). This will determine
   the name of your image, and you will push / pull from it. Your image
   name will be `quay.io/<username>/<repository-name>`.
3. Go to your account settings (under your name in the top right), and
   select the 'Robot Accounts' option on the left menu.
4. Click 'Create Robot account', give it a memorable name (such as
   `<hub-name>_image_builder`) and click 'Create'
5. In the next screen, select the repository you just created in step (2),
   and give the robot account `Write` permission to the repository.
6. Once done, click the name of the robot account again. This will give you
   its username and password.
7. Create these [GitHub secrets](https://docs.github.com/en/actions/reference/encrypted-secrets)
   for your repository with the credentials from the robot account:
   1. `QUAY_USERNAME`: user name of the robot account
   2. `QUAY_PASSWORD`: password of the robot account
   
8. Use the following config for your github action.
   ```yaml
   name: Build container image

   on: [push]

   jobs:
     build:
       runs-on: ubuntu-latest
       steps:

       - name: checkout files in repo
         uses: actions/checkout@main

       - name: update jupyter dependencies with repo2docker
         uses: jupyterhub/repo2docker-action@master
         with: # make sure username & password/token matches your registry
           DOCKER_USERNAME: ${{ secrets.QUAY_USERNAME }}
           DOCKER_PASSWORD: ${{ secrets.QUAY_PASSWORD }}
           DOCKER_REGISTRY: "quay.io"
           IMAGE_NAME: "<quay-username>/<repository-name>"

   ```

## Push Repo2Docker Image To Amazon ECR

1. Login to [Amazon AWS Console](https://console.aws.amazon.com/)
2. [Create an individual IAM user](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#create-iam-users) who's access key will be used by the GitHub Actions. Make sure the user has permissions to make calls to the Amazon ECR APIs and to push/pull images to the repositories you need.
Checkout and follow [Amazon IAM best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) for the AWS credentials used in GitHub Actions workflows.
3. Create a new private [repository](https://us-east-2.console.aws.amazon.com/ecr/create-repository). This will determine the name of your image, and you will push / pull from it. Your image name will be `<aws-account-id>.dkr.ecr.<aws-region>.amazonaws.com/<username>/<repository-name>`.
4. Go to the IAM dashboard, ['Users' section](https://console.aws.amazon.com/iamv2/home#/users) and click on the username created at `Step 2`.
Click on 'Security credentials' tab, right below the 'Summary' section. In the 'Access keys' section, click on the 'Create access key' button. 
Once done, it will give you an 'Access key ID' and the 'Secret access key'.
5. Create these [GitHub secrets](https://docs.github.com/en/actions/reference/encrypted-secrets)
   for your repository with the credentials from the robot account:
   1. `AWS_ACCESS_KEY_ID`: access key id of the IAM user
   2. `AWS_SECRET_ACCESS_KEY`: secret access key of the IAM user

6. Use the following config for your github action.
   ```yaml
   name: Build container image

   on: [push]

   jobs:
     build:
       runs-on: ubuntu-latest
       env:
         DOCKER_CONFIG: $HOME/.docker
       steps:
       - name: checkout files in repo
         uses: actions/checkout@main

       - name: Configure AWS Credentials
         uses: aws-actions/configure-aws-credentials@v1
         with:
           aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
           aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
           aws-region: <region>

       - name: Login to Amazon ECR
         id: login-ecr
         uses: aws-actions/amazon-ecr-login@v1


       - name: Update jupyter dependencies with repo2docker
         uses: jupyterhub/repo2docker-action@master
         with:
           DOCKER_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
           IMAGE_NAME: "<aws-username>/<repository-name>"

   ```

## Push Repo2Docker Image To Google Container Registry

1. Login to [Google Cloud Console](https://console.cloud.google.com)
2. Create (or use an existing) Google Cloud Project with the billing activated. This will be the place where the registry hosting the repo2docker image will live.
3. Make sure [`Container Registry API`](https://console.cloud.google.com/apis/library/containerregistry.googleapis.com) is enabled for this project.
4. The repository will be created automatically once the first image is pushed. Your image name will be `grc.io/<gcp-project-id>/<repository-name>`.
5. Create a Service Account to authenticate the calls made by GitHub Actions to our GCP project:
   - In the Cloud Console, go to the [Service Accounts page](https://console.cloud.google.com/iam-admin/serviceaccounts).
   - Make sure the right project is selected in the drop-down menu above.
   - Click on [`Create Service Account`](https://console.cloud.google.com/iam-admin/serviceaccounts/create)
   - Enter a service account name — give it a memorable name (such as `<hub-name>_image_builder`).
   - Grant this service account access to project. As a best practice, grant it only the minimum permissions: `Cloud Run Admin`, `Service Account User`, and `Storage Admin`.
6. Click on the service account's name you just created and select the `Keys` tab. Click on the `ADD KEY` button, select `Create new key`, then create a JSON key type. The private key will be saved to your computer. Make sure to store it somewhere secure!
7. Create these [GitHub secrets](https://docs.github.com/en/actions/reference/encrypted-secrets)
   for your repository with the credentials from the robot account:
   1. `GCP_SA_KEY`: the private key of the service account created in the previous step
   2. `GCP_PROJECT_ID`: the id of the Google Cloud Project

8. Use the following config for your github action.
   ```yaml
   name: Build container image

   on: [push]

   jobs:
     build:
       runs-on: ubuntu-latest
       env:
         DOCKER_CONFIG: $HOME/.docker

       steps:
       - name: checkout files in repo
         uses: actions/checkout@main

       - name: Login to GCR
         uses: docker/login-action@v1
         with:
           registry: gcr.io
           username: _json_key
           password: ${{ secrets.GCP_SA_KEY }}

       - name: Update jupyter dependencies with repo2docker
         uses: jupyterhub/repo2docker-action@master
         with:
           DOCKER_REGISTRY: gcr.io
           IMAGE_NAME: ${{ secrets.GCP_PROJECT_ID }}/<repository-name>
     ```

## Push Repo2Docker Image To Google Artifact Registry

1. Login to [Google Cloud Console](https://console.cloud.google.com)
2. Create (or use an existing) Google Cloud Project with the billing activated. This will be the place where the registry hosting the repo2docker image will live.
3. Make sure [`Artifact Registry API`](https://console.cloud.google.com/apis/library/artifactregistry.googleapis.com) is enabled for this project.
4. Create a new [artifact repository](https://console.cloud.google.com/artifacts/create-repo). This will determine the name and location of your image. Your image name will be `<location>-docker.pkg.dev/<gcp-project-id>/<repository-name>`
5. Create a Service Account to authenticate the calls made by GitHub Actions to our GCP project:
   - In the Cloud Console, go to the [Service Accounts page](https://console.cloud.google.com/iam-admin/serviceaccounts).
   - Make sure the right project is selected in the drop-down menu above.
   - Click on [`Create Service Account`](https://console.cloud.google.com/iam-admin/serviceaccounts/create)
   - Enter a service account name — give it a memorable name (such as `<hub-name>_image_builder`).
   - Grant this service account access to project. As a best practice, grant it only the minimum permissions: `Cloud Run Admin`, `Service Account User`, `Storage Admin`, `Artifact Registry Repository Administrator`.
6. Click on the service account's name you just created and select the `Keys` tab. Click on the `ADD KEY` button, select `Create new key`, then create a JSON key type. The private key will be saved to your computer. Make sure to store it somewhere secure!
7. Create these [GitHub secrets](https://docs.github.com/en/actions/reference/encrypted-secrets)
   for your repository with the credentials from the robot account:
   1. `GCP_SA_KEY`: the private key of the service account created in the previous step
   2. `GCP_PROJECT_ID`: the id of the Google Cloud Project

8. Use the following config for your github action.
   ```yaml
   name: Build container image

   on: [push]

   jobs:
     build:
       runs-on: ubuntu-latest
       env:
         DOCKER_CONFIG: $HOME/.docker

       steps:
       - name: checkout files in repo
         uses: actions/checkout@main

       - name: Login to GAR
         uses: docker/login-action@v1
         with:
           registry: <location>-docker.pkg.dev
           username: _json_key
           password: ${{ secrets.GCP_SA_KEY }}

       - name: Update jupyter dependencies with repo2docker
         uses: jupyterhub/repo2docker-action@master
         with:
           DOCKER_REGISTRY: <location>-docker.pkg.dev
           IMAGE_NAME: ${{ secrets.GCP_PROJECT_ID }}/<repository-name>

     ```

## Push Repo2Docker Image To Azure Container Registry

1. Login to [Azure Portal](https://portal.azure.com/)
2. Create a new [container registry](https://portal.azure.com/#create/Microsoft.ContainerRegistry). This will determine the name of your image, and you will push / pull from it. Your image name will be `<container-registry-name>.azurecr.io/<repository-name>`.
3. Go to `Access Keys` option on the left menu.
4. Enable `Admin user` so you can use the registry name as username and admin user access key as password to docker login to your container registry.
5. Create these [GitHub secrets](https://docs.github.com/en/actions/reference/encrypted-secrets)
   for your repository with the credentials from the robot account:
   1. `ACR_USERNAME`: the registry name
   2. `ACR_PASSWORD`: the access key of the admin user

6. Use the following config for your github action.
   ```yaml
   name: Build container image

   on: [push]

   jobs:
     build:
       runs-on: ubuntu-latest

       steps:
       - name: checkout files in repo
         uses: actions/checkout@main

       - name: Update jupyter dependencies with repo2docker
         uses: jupyterhub/repo2docker-action@master
         with:
           DOCKER_USERNAME: ${{ secrets.ACR_USERNAME }}
           DOCKER_PASSWORD: ${{ secrets.ACR_PASSWORD }}
           DOCKER_REGISTRY: <container-registry-name>.azurecr.io
           IMAGE_NAME: <repository-name>

   ```

## Push Repo2Docker Image To Other Registries

If the docker registry accepts a credentials to be passed as a username and password string, you can do it like this.

```yaml
name: Build Notebook Container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with: # make sure username & password/token matches your registry
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        DOCKER_REGISTRY: "gcr.io"
```

If the docker registry doesn't credentials to be passed as a username and password strong, or if you want to do it in another way, you can configure credentials to the docker registry ahead of time instead.  Below is an incomplete example doing that.

```yaml
name: Build Notebook Container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    # TODO: add a step here to setup credentials to push to your
    #       docker registry before running the repo2docker-action

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_REGISTRY: your-registry.example.org
        IMAGE_NAME: your-image-name
```

## Change Image Name

When you do not provide an image name your image name defaults to `DOCKER_USERNAME/GITHUB_REPOSITORY_NAME`.  For example if the user [`hamelsmu`](http://www.github.com/hamelsmu) tried to run this Action from this repo, it would be named `hamelsmu/repo2docker-action`.  However, sometimes you may want a different image name, you can accomplish by providing the `IMAGE_NAME` parameter as illustrated below:

```yaml
name: Build Notebook Container
on: [push]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:

    - name: checkout files in repo
      uses: actions/checkout@main

    - name: update jupyter dependencies with repo2docker
      uses: jupyterhub/repo2docker-action@master
      with:
        DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
        DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
        IMAGE_NAME: "hamelsmu/my-awesome-image" # this overrides the image name
```

## Test Image Build

You might want to only test the image build withtout pushing to a registry, for example to test a pull request. You can do this by specifying any value for the `NO_PUSH` parameter:

```yaml
name: Build Notebook Container
on: [pull_request]
jobs:
  build-image-without-pushing:
    runs-on: ubuntu-latest
    steps:  
    - name: Checkout PR
      uses: actions/checkout@v2
      with:
        ref: ${{ github.event.pull_request.head.sha }}

    - name: test build
      uses: jupyterhub/repo2docker-action@master
      with:
        NO_PUSH: 'true'
        IMAGE_NAME: "hamelsmu/repo2docker-test"
```

_When you specify a value for the `NO_PUSH` parameter, you can omit the otherwhise mandatory parameters `DOCKER_USERNAME` and `DOCKER_PASSWORD`._

# Contributing To repo2docker-action

See the [Contributing Guide](./CONTRIBUTING.md).
