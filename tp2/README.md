# TP2 Github Action
## Setup GitHub Actions

__Q: What are testcontainers?__
test containers is an opensource library providing containers for testing purposes. It is used in several programming langages.

- Github action configuration "main.yml":
```yaml
name: CI devops 2025
on:
  push:
    branches:
        - master
  pull_request:

jobs:
  test-backend: 
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v2.5.0
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'corretto'
      - name: Build and test with Maven
        run: mvn clean verify
        working-directory: tp1/backend/simple-api-student
```
This is the configuration file for one workflow with one job: building and testing the app. This job is done in three steps:
- a checkout action so the workflow can acces the configured branch.
- a java setup action, to download and install the configured version of java on the runner
- a build action to test the programm. The "working directory" directive gives the location of the project to build and test

## First steps into the CD World

- new job added to the "main.yml" file to build and push project docker images:
```yaml
build-and-push-docker-image:
    needs: test-backend
    permissions:
      packages: write
      contents: read
      attestations: write
      id-token: write
    runs-on: ubuntu-22.04
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Build image and push backend
        uses: docker/build-push-action@v6
        with:
          push: ${{ github.ref == 'refs/heads/master' }}
          context: tp1/backend/simple-api-student
          tags:  ${{ secrets.DOCKER_USERNAME  }}/tp-devops-simple-api:latest
      - name: Build image and push database
        uses: docker/build-push-action@v6
        with:
          push: ${{ github.ref == 'refs/heads/master' }}
          context: tp1/pgsql
          tags:  ${{ secrets.DOCKER_USERNAME  }}/pgsql-simple-api:latest
      - name: Build image and push httpd
        uses: docker/build-push-action@v6
        with:
          push: ${{ github.ref == 'refs/heads/master' }}
          context: tp1/http_server
          tags:  ${{ secrets.DOCKER_USERNAME  }}/http-simple-api:latest
```
With the directive "needs", the new job is run after the "test-backend" job. This one is composed of 5 steps:
- The usual checkout action so the workflow as acces to the workspace
- docker login step: important for the push steps. We can't publish on docker hub if we are not logged in
- Three steps to build and push the different docker images. We put in the "push" option a condition to be sure that we push in the "master" (main) branch

All the secrets are saved on the github repository. To add secrets to repository: repository setting --> secret and variables --> Actions --> New repository secret

__Q: For what purpose do we need to push docker images?__

It is important to push docker images once the different tests are finished to publish the new version of the images. Of course we must be sure that our different jobs test all aspects of our app.

## Setup Quality Gate

- once we created our accont on SonarCloud and that we have setup our project, we need to modify the build and test step:
```yaml
steps:
    - name: Build and test with Maven
      run: mvn -B verify sonar:sonar -Dsonar.projectKey=qwertyjuju_DevOps-CPE -Dsonar.organization=qwertyjuju -Dsonar.host.url=https://sonarcloud.io -Dsonar.login=${{ secrets.SONAR_TOKEN }}
      working-directory: tp1/backend/simple-api-student
```
Each time a new push is done, sonarcloud will do a code verification.