# QAA SDL3 - Module 5 - Exercise 4
This is a reference solution for QAA SDL3 Module 5 - Workflow Design
The objective is to design a workflow to build and deploy a web application with the architecture shown below:
![arch-dia](https://github.com/agray998/QAA-Module5-Workflow-Design-Solution/blob/main/aws-dia.drawio%20(1).png)
The design should consider:
* What needs to be done to build and deploy such an app from source code?
* What actions might be used to facilitate this?
* What sensitive information is required, and how should it be stored?

## Building/Deploying Back-end
The workflow will probably start by building and deploying the updated back-end. The build process should be familiar - after checking out the source code, configure a JDK installation, navigate to the location of the pom.xml file and run:
```shell
mvn clean test # run the unit tests
mvn clean package # build the jar file
```
To deploy the resulting jar to Elastic Beanstalk, the easiest way would be to use the [Beanstalk Deploy module](https://github.com/einaregilsson/beanstalk-deploy). This will require that you have AWS CLI credentials available; these should be stored securely as secrets.

## Building/Deploying Front-end
To build and deploy the front-end, you will first need to install any dependencies (can be installed via npm, which should be available by default on the runner), and compile your project to produce standalone js files using `ncc` or a similar tool.  

Once you have generated the static files which are to be served as the front-end, these can be uploaded to the S3 bucket hosting the front-end either via the AWS CLI directly, or via one of the many S3 sync actions available - [this one](https://github.com/marketplace/actions/s3-sync) for example. This will also require AWS CLI credentials, as before these should be stored as secrets. 

## Solution Template
```yaml
name: build/deploy 3-tier web-app

env:
    AWS_S3_BUCKET: mtwa-frontend-3r32nenevnwmtrhnm # remember S3 bucket names have to be unique

on:
    push:
        branches: ["main"]
    workflow_dispatch:

jobs:
    build_deploy_back:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v3
        - uses: actions/setup-java@v3
          with:
            java-version: '11'
            distribution: 'temurin'
        - name: run tests and build
          run: |
            cd backend/
            mvn clean test | tee test.log
            mvn clean package
        - uses: einaregilsson/beanstalk-deploy@v21
          with:
            aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            application_name: MultiTierWebApp
            environment_name: MultiTierWebApp-Environment
            version_label: 123
            region: eu-west-2
            deployment_package: target/MTWA-BACKEND-0.0.1.jar
    
    build_deploy_front:
        runs-on: ubuntu-latest
        steps:
        - uses: actions/checkout@v3
        - name: build dist files
          run: |
            cd frontend/
            npm install
            ncc build main.js -o dist
        - uses: jakejarvis/s3-sync-action@master
          with:
            args: --acl public-read --follow-symlinks --delete
          env:
            AWS_S3_BUCKET: ${{ env.AWS_S3_BUCKET }}
            AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
            AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            AWS_REGION: 'eu-west-2'
            SOURCE_DIR: 'dist'
```

## Further Developments
As a point of interest, this workflow could be developed further to make it even more useful. Some ideas to think about:
<details>
<summary>How could the workflow be used to configure the services needed to host the solution?</summary>
AWS CloudFormation can be used to configure resources and services from templates defined, like the workflow itself, as YAML files. AWS provides an <a href="https://github.com/aws-actions/aws-cloudformation-github-deploy">action</a> for deploying CloudFormation stacks which could be incorporated into the workflow.
</details>
