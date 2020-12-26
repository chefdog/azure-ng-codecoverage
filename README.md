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
For this example, we will use a blank angular project. I have used this guide on more professional angular projects also. 
First I have created a new github repository and did a clone locally.
But in this case, lets just setup a new angular 11 project:

'ng new cfg-ng' and after lets go into the project dir and run 'ng serve', 'ng test' and 'ng lint', just to see if everything works.

## 4. Additional packages
We need some extra packages in order to make things work, lets install them:
- npm install puppeteer --save-dev
- npm install karma-junit-reporter --save-dev
- npm install jasmine-reporters --save-dev



