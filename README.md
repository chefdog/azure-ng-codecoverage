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
```
npm install puppeteer --save-dev
npm install karma-junit-reporter --save-dev
npm install jasmine-reporters --save-dev
```


## 5. Azure DevOps Project and YML

If you log into Azure, create a new project. Make sure the Version Control of the project is GIT (under advanced).
Most of the time, I select the scrum option for work item proces.

Next we need to create a pipeline (a build pipeline that is). Click on create pipeline and select Github if your code is located at Github.
Azure DevOps will automatically hook up the repo to the pipeline (after authentication and authorization).

Next Azure will ask to configure the pipeline. Select the 'Starter Pipeline', it will create a basic YML file.
Save and run the pipeline.

The YML file should be part of your code base and will be pushed into Git, so be sure to fetch and pull.
In order to modify the YML file, you could do that in Azure DevOps or locally in VS Code.
The most easy way is to use Azure DevOps. You can drag and drap predefined tasks into the YML file.

### 5.1. trigger, pool and vars

First part of the YML file, is the setup. The default template had one trigger and that is main. We want to have both main and develop as triggers in order
to start the build pipeline. The pool we will not modify. Next we have some variables, one for build configuration and one for angular's build configuration.

```
trigger:
  branches:
    include:
      - develop
      - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  ngBuildConfiguration: '--prod'
  ngWorkingDir: 'cfg-ng'
```

### 5.2 Steps and tasks
First, remove the 2 example script tasks. Basically all code after steps.
Next create a new task, based on the NPM task template. It is just the default npm install task.
The only change we need is the workingDir.
```
steps:
- task: Npm@1
  displayName: 'npm install'
  inputs:
    command: 'install'
    workingDir: '$(ngWorkingDir)'
```

In order to build the angular project we add another npm task, but this time it is a custom task.
It basically executes the build command located in the package.json file with the additional param `--prod`
```
- task: Npm@1
  inputs:
    command: 'custom'
    workingDir: '$(ngWorkingDir)'
    customCommand: 'run build -- $(ngBuildConfiguration)'
```

After building the angular source code, we want to publish the result. We need 2 tasks for this.
Just type publish in the search field and drag n drop 'Publish Pipeline Artifacts' and 'Publish Build Artifactd' onto the YML.

```
- task: PublishPipelineArtifact@0
  inputs:
    artifactName: angularPipelineArtifactProd
    targetPath: 'src/CustoMassWeb/dist'

- task: PublishBuildArtifacts@1
  inputs:
    PathtoPublish: 'src/CustoMassWeb/dist'
    ArtifactName: 'angularBuildArtifactProd'
```

### 5.3 Test, Code coverage steps and tasks 

#### 5.3.1
Before we create the next section of the pipeline, we need to change the karma.conf.js first.
In the orignal article 'ng test' runs chrome headless. I wanted to leave the default 'ng test' working and have an extra option
to run the headless version. We need the headless version to hook up the proces in Azure. But locally you may want to use the normal 'ng test',
to debug your test code.

There are 7 modifications needed in the karma.conf.js file:
1. add puppeteer
2. add karma reporter
3. add covertura reporter
4. add junit reporter
5. add junitreporter with output dir
6. add Chromeheadless
7. add custom launchers

So this should be the result:

```
module.exports = function (config) {
  // 1. add puppeteer
  const puppeteer = require('puppeteer');
  process.env.CHROME_BIN = puppeteer.executablePath();

  config.set({
    basePath: '',
    frameworks: ['jasmine', '@angular-devkit/build-angular'],
    plugins: [
      require('karma-jasmine'),
      require('karma-chrome-launcher'),
      require('karma-jasmine-html-reporter'),
      require('karma-coverage'),
      require('@angular-devkit/build-angular/plugins/karma'),
      require('karma-junit-reporter') // 2. add karma reporter
    ],
    client: {
      jasmine: {
      },
      clearContext: false // leave Jasmine Spec Runner output visible in browser
    },
    jasmineHtmlReporter: {
      suppressAll: true // removes the duplicated traces
    },
    coverageReporter: {
      dir: require('path').join(__dirname, './coverage/cfg-ng'),
      subdir: '.',
      reporters: [
        { type: 'html' },
        { type: 'text-summary' },
        { type: 'cobertura'} // 3. add covertura reporter
      ]
    },
    reporters: ['progress', 'kjhtml', 'junit'], // 4. add junit reporter
    junitReporter: {
      outputDir: '../junit' // 5. add junitreporter with output dir
    }, 
    port: 9876,
    colors: true,
    logLevel: config.LOG_INFO,
    autoWatch: true,
    browsers: ['Chrome', 'ChromeHeadless'], // 6. add Chromeheadless
    // 7. add custom launchers
    customLaunchers:{
      ChromeHeadless: {
        base: 'Chrome',
        flags: [
          '--no-sandbox',
          '--headless',
          '--disable-gpu',
          '--remote-debugging-port=9222'
        ]
      }
    },
    singleRun: false,
    restartOnFileChange: true
  });
};
```

Also, modify the package.json scripts section:
```
"scripts": {
    "ng": "ng",
    "start": "ng serve",
    "build": "ng build",
    "install-puppeteer": "cd node_modules/puppeteer && npm run install",
    "test-headless": "npm run install-puppeteer && ng test --watch=false --code-coverage --browsers=ChromeHeadless",
    "test": "ng test",
    "lint": "ng lint",
    "e2e": "ng e2e"
  },
```



#### 5.3.2
Lets continue with the pipeline, we need to have a cleanup task first.
Create a new task 'Delete files' or just search in azure and add the task.
Modify the source folder and contents.

And add the actual test task. The test task is another NPM custom task.
It hooks up the command located in the package.json

```
- task: DeleteFiles@1
  displayName: 'Delete JUnit files'
  inputs:
    SourceFolder: 'cfg-ng/junit'
    Contents: 'TESTS*.xml'

- task: Npm@1
  displayName: 'Test angular'
  inputs:
    command: 'custom'
    workingDir: 'cfg-ng'
    customCommand: 'run test-headless'

```

The last 2 tasks are there to publish the results. Again 2 publish tasks:

```
- task: PublishCodeCoverageResults@1
  displayName: 'Publish code coverage Angular'
  condition: succeededOrFailed()
  inputs:
    codeCoverageTool: 'Cobertura'
    summaryFileLocation: 'cfg-ng/coverage/cobertura-coverage.xml'
    reportDirectory: 'cfg-ng/coverage'
    failIfCoverageEmpty: false

- task: PublishTestResults@2
  displayName: 'Publish Angular test results'
  condition: succeededOrFailed()
  inputs:
    searchFolder: $(System.DefaultWorkingDirectory)/cfg-ng/junit
    testResultsFormat: 'JUnit'
    testResultsFiles: '**/TEST-*.xml'
    failTaskOnFailedTests: false
    testRunTitle: 'Angular'
```

### 6. gitignore
If you run the headless test locally, I think you should not push them into git, so add something like this:
```
# JUnit
/cfg-ng/junit
/cfg-ng/junit/*.xml
```
