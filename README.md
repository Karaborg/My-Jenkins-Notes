# Welcome
This is me taking notes on GitHub while learning `Jenkins`. I will probably be going to need this document when I forget how it works. If you find something useful, be my guest. I will try to update this whenever I study.

## Setup
`docker run -p 8080:8080 -p 50000:50000 -v "C:\Users\<PATH>\var\jenkins_home\:/var/jenkins_home" -d --name jenkins jenkins/jenkins:lts`

## Building a NodeJS App
- Install NodeJS from `Dashboad > Manage Jenkins > Manage Plugins > Available plugins`
- Restart Jenkins
- Go to `Dashboard > Manage Jenkins > Global Tool Configuration` and add NodeJS
- Choose **Restart Afterward**
- Create a new job from the Dashboard (freestyle)
- Under **Source Code Management**, choose **Git** and paste the git URL as `Repository URL`
- Add build step with **execute shell** which is `npm install`
- Under **Build Environment**, enable `Provide Node & nmp bin/ folder to PATH` and select NodeJS 
- Click on **save** and then **build** 
- You can monitor the steps in **Console Output**

- Go to `Dashboard > Manage Jenkins > Global Tool Configuration` and add `CloudBees Docker Build and Publish`
- choose **Restart Afterward**
- 

## Infrastructure as Code & Automation
### Jenkins Job DSL
You can create jobs just by a single groovy file using **Jenkins DSL**
Example:
```
job('<NAME_OF_THE_JOB>') {
    scm {
        git('<GIT_REPOSITORY>') { node ->
            node / gitConfigName('<GIT_USER>')
            node / gitConfigEmail('<GIT_E-MAIL_ADDRESS>')
        }
    }

    triggers {
        scm('H/5 * * * *') // How many times do you want to build it
    }

    wrappers {
        nodejs('nodejs') // this is the name of the NodeJS installation in
                         // Manage Jenkins > Configure Tools > NodeJS Installations > Name
    }

    steps {
        shell("npm install")
    }
}
```
- Install `Job DSL` from `Dashboad > Manage Jenkins > Manage Plugins > Available plugins`
- Restart Jenkins
- Create new item (freestyle)
- If your groovy file is on GIT, gave the project GIT URL as **Repository**
- Under `Build`, add `Process Job DSLs`
- Under `Look on Filesystem`, paste the groovy file path from the project path
- **Save** and **build** the job. It will ***fail*** 
- Go to `Dashboard > Manage Jenkins > In-Process Script Approval`
- You should be able to see your groovy script. **Approve** it by clicking the approve button on top
- Build again

You can also create a job that will build the docker image and pushes it to **Docker Hub**
Example:
```
job('<NAME_OF_THE_JOB>') {
    scm {
        git('<GIT_REPOSITORY>') {  node -> // is hudson.plugins.git.GitSCM
            node / gitConfigName('<GIT_USER>')
            node / gitConfigEmail('<GIT_E-MAIL_ADDRESS>')
        }
    }
    triggers {
        scm('H/5 * * * *')
    }
    wrappers {
        nodejs('nodejs') // this is the name of the NodeJS installation in 
                         // Manage Jenkins -> Configure Tools -> NodeJS Installations -> Name
    }
    steps {
        dockerBuildAndPublish {
            repositoryName('<DOCKER_HUB_REPOSITORY_PATH>')
            tag('${GIT_REVISION,length=9}')
            registryCredentials('dockerhub')
            forcePull(false)
            forceTag(false)
            createFingerprints(false)
            skipDecorate()
        }
    }
}
```
> For more information about Docker API's, check out [Jenkins Job DSL Plugin](https://jenkinsci.github.io/job-dsl-plugin/#method/javaposse.jobdsl.dsl.helpers.step.StepContext.dockerBuildAndPublish) page.

- install `Job DSL` from `Dashboad > Manage Jenkins > Manage Plugins > Available plugins`
- Restart Jenkins
- Create new item (freestyle)
- If your groovy file is on GIT, gave the project GIT URL as **Repository**
- Under `Build`, add `Process Job DSLs`
- Under `Look on Filesystem`, paste the groovy file path from the project path
- **Save** and **Exit**.
- Go to `Dashboard > Manage Jenkins > Credentials`
- Click on the **downward arrow** next to `(global)` proved by `System`
- `Kind`: `Username With Password`
- `Scope`: `Global`
- Enter your **username** and **password**
- `Id`: `dockerhub`
- Build the job. it will fail 
- Go to `Dashboard > Manage Jenkins > In-Process Script Approval`
- You should be able to see your groovy script. **Approve** it by clicking the approve button on top
- Build again

## Jenkins Pipelines
`The Jenkins Pipelines` is a job type, you can create a Jenkins pipeline job that will handle the **build**/**test**/**deployment** of one project.

Example:
```
node { // this project can run on any node
    def mvnHome

    stage('Preparation') {
        git '<GIT_REPOSITORY_URL>'
        // Get the Maven tool.
        // ** NOTE: This 'M3' Maven tool must be configured in the global configuration.
        mvnHome = tool 'M3'
    }

    stage('Build') {
        // Run the maven build
        if (isUnix()) {
            sh "'${mvnHome}/bin/mvn' -Dmaven.test.failure.ignore clean package"
        } else {
            bat(/"${mvnHome}\bin\mvn" -Dmaven.test.failure.ignore clean package/)
        }
    }

    stage('Results') {
        junit '**/target/surefire-reports/TEST-*.xml'
        archive 'target/*.jar'
    }
}
```

Example:
```
node {
    def commit_id
    
    stage('Preparation') {
        checkout scm // will make the git repository files available for jenkins
        sh "git rev-parse --short HEAD > .git/commit-id" // gets the commit id
        commit_id = readFile('.git/commit-id').trim()
    }
    
    stage('test') {
        nodejs(nodeJSInstallationName: 'nodejs') {
            sh 'npm install --only=dev'
            sh 'npm test'
        }
    }
    
    stage('docker build/push') {
        docker.withRegistry('https://index.docker.io.v1/', 'dockerhub') { // make sure you have dockerhub credentials on jenkins with the name of dockerhub
            def app = docker.build("<DOCKER_HUB_REPOSITORY>:${commit-id}", '.').push()
        }
    }
}
```

- Create a new item
- Choose Pipeline
- Under Pileline, choose Pipeline script from SCM
- Under SCM, choose Git
- Give the Git Repository URL as Repository URL
- Give the Jenkins file path as Script Path (from the root folder of the project)
- Save and build

You can also use Docker Pipeline plugin
Example:
```
node {
    def commit_id
    
    stage('Preparation') {
        checkout scm
        sh "git rev-parse --short HEAD > .git/commit-id"
        commit_id = readFile('.git/commit-id').trim()
    }
    
    stage('test') {
        def myTestContainer = docker.image('node:4.6')
        
        myTestContainer.pull()
        myTestContainer.inside {
            sh 'npm install --only=dev'
            sh 'npm test'
        }
    }
    
    stage('test with a DB') {
        def mysql = docker.image('mysql').run("-e MYSQL_ALLOW_EMPTY_PASSWORD=yes")
        def myTestContainer = docker.image('node:4.6')
        
        myTestContainer.pull()
        myTestContainer.inside("--link ${mysql.id}:mysql") { // using link, mysql will be available at host: mysql, port:3306
            sh 'npm install --only=dev'
            sh 'npm test'
        }
        mysql.stop()
    }
    
    stage('docker build/push') {
        docker.withRegistry('https://index.docker.io.v1/', 'dockerhub') { // make sure you have dockerhub credentials on jenkins with the name of dockerhub
            def app = docker.build("<DOCKER_HUB_REPOSITORY>:${commit-id}", '.').push()
        } 
    }
}
```

- Create a new item
- Choose Pipeline
- Under Pileline, choose Pipeline script from SCM
- Under SCM, choose Git
- Give the Git Repository URL as Repository URL
- Give the Jenkins file path as Script Path (from the root folder of the project)
- Save and build

## Jenkins Integrations
### Email Integration















## Advanced Jenkins Usage

## End


















