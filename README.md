# azure-ng-codecoverage
How to on integration of ng test. ng lint in azure devops 2020

## 1. Credits 
Based on https://www.olivercoding.com/2020-01-02-angular-azure-devops/

## 2. Requirements
- github account
- An Azure devops account and project
- npm installed locally
- angular cli installed locally : https://angular.io/guide/setup-local
- Visual Studio Code

## 3. Project setup
For this example, we will use a blank angular project. 
First I have created a new github repository and did a clone locally.
But in this case, lets just setup a new angular 11 project:

`ng new cfg-ng` and after lets go into the project dir and run `ng serve`, `ng test` and `ng lint`, just to see if everything works.

## 4. Additional packages
We need some extra packages in order to make things work, lets install them:
`npm install puppeteer --save-dev`
`npm install karma-junit-reporter --save-dev`
`npm install jasmine-reporters --save-dev`


## 5. Azure DevOps Project and YML

If you log into Azure, create a new project. Make sure the Version Control of the project is GIT (under advanced).
Most of the time, I select the scrum option for work item proces.

Next we need to create a pipeline (a build pipeline that is). Click on create pipeline and select Github if your code is located at Github.
Azure DevOps will automatically hook up the repo to the pipeline (after authentication and authorization).

Next Azure will ask to configure the pipeline. Select the 'Starter Pipeline', it will create a basic YML file.
Save and run the pipeline.

The YML file should be part of your code base, so be sure to fetch and pull.

### 5.1. trigger, pool and vars

First part of the YML file, is the setup. 

trigger:
  branches:
    include:
      - develop
      - master

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  ngBuildConfiguration: '--prod'

### 5.2 Steps and tasks

steps:
- task: Npm@1
  displayName: 'npm install'
  inputs:
    command: 'install'
    workingDir: 'src/CustoMassWeb'

- task: Npm@1
  inputs:
    command: 'custom'
    workingDir: 'src/CustoMassWeb'
    customCommand: 'run build -- $(ngBuildConfiguration)'

- task: PublishPipelineArtifact@0
  inputs:
    artifactName: angularPipelineArtifactProd
    targetPath: 'src/CustoMassWeb/dist'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: 'src/CustoMassWeb/dist'
    ArtifactName: 'angularBuildArtifactProd'

- task: DeleteFiles@1
  displayName: 'Delete JUnit files'
  inputs:
    SourceFolder: 'src/CustoMassWeb/junit'
    Contents: 'TESTS*.xml'
- task: Npm@1
  displayName: 'Test angular'
  inputs:
    command: 'custom'
    workingDir: 'src/CustoMassWeb'
    customCommand: 'run test -- --watch=false --code-coverage'
- task: PublishCodeCoverageResults@1
  displayName: 'Publish code coverage Angular'
  condition: succeededOrFailed()
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: 'src/CustoMassWeb/coverage/cobertura-coverage.xml'
    reportDirectory: 'src/CustoMassWeb/coverage'
    failIfCoverageEmpty: false
- task: PublishTestResults@2
  displayName: 'Publish Angular test results'
  condition: succeededOrFailed()
  inputs:
    searchFolder: $(System.DefaultWorkingDirectory)/src/CustoMassWeb/junit
    testResultsFormat: 'JUnit'
    testResultsFiles: '**/TEST-*.xml'
    failTaskOnFailedTests: false
    testRunTitle: 'Angular'


