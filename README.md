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

- Create a **new item**
- Choose `Pipeline`
- Under `Pipeline`, choose `Pipeline script from SCM`
- Under `SCM`, choose `Git`
- Give the **Git Repository URL** as `Repository URL`
- Give the **Jenkins file path** as `Script Path` (from the root folder of the project)
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

- Create a **new item**
- Choose `Pipeline`
- Under `Pipeline`, choose `Pipeline script from SCM`
- Under `SCM`, choose `Git`
- Give the **Git Repository URL** as `Repository URL`
- Give the **Jenkins file path** as `Script Path` (from the root folder of the project)
- Save and build

## Jenkins Integrations
### Email Integration
- Install `Email Extension Plugin` from `Dashboad > Manage Jenkins > Manage Plugins` if it's not installed already
- Go to `Manage Jenkins > Configure System` and you can configure your Email notifications. (SMTP server configurations)
- After the configuration, you can make specific configurations on your **Jenkins file**
Example:
```
node {

  // config 
  def to = emailextrecipients([
          [$class: 'CulpritsRecipientProvider'],
          [$class: 'DevelopersRecipientProvider'],
          [$class: 'RequesterRecipientProvider']
  ])

  // job
  try {
    stage('build') {
      println('so far so good...')
    }
    stage('test') {
      println('A test has failed!')
      sh 'exit 1'
    }
  } catch(e) {
    // mark build as failed
    currentBuild.result = "FAILURE";
    // set variables
    def subject = "${env.JOB_NAME} - Build #${env.BUILD_NUMBER} ${currentBuild.result}"
    def content = '${JELLY_SCRIPT,template="html"}'

    // send email
    if(to != null && !to.isEmpty()) {
      emailext(body: content, mimeType: 'text/html',
         replyTo: '$DEFAULT_REPLYTO', subject: subject,
         to: to, attachLog: true )
    }

    // mark current build as a failure and throw the error
    throw e;
  }
}
```
- Create a `pipeline` job
- Under `Pipeline`, choose `Pipeline script from SCM`
- Under `SCM`, choose `Git`
- Give the **Git Repository URL** as `Repository URL`
- Give the **Jenkins file path** as `Script Path` (from the root folder of the project)
- You can also `Poll SCM` to set timer ("H/5 * * * *" would run every 5 minutes)
- Save and build

### Slack Integration
- Install `Slack Notification Plugin` from `Dashboad > Manage Jenkins > Manage Plugins`
- Go to Manage `Jenkins > Configure System` and you can configure your `Global Slack Notifier Settings`
- Make sure you have **credendial ID** for `Integration Token Credential ID`

### GitHub / Bitbucket Integration

#### GitHub Integration
- Install `GitHub Branch Source Plugin` from `Dashboad > Manage Jenkins > Manage Plugins`
- Go to `Manage Jenkins > Configure System` and you can configure your `GitHub Branch Source Plugin`
- Create a new item as `GitHub Organization`
- Under `Project`, add **credentials**.
 - To add GitHub credentials, go to `GitHub > Settings > Repositories > Generate New Token`
 - Select `repo` as `Scopes` and create the token
- Make sure you also add **owner**
- Since we are using Gradle as compiler, we are going to create a `.gradle` file inside our Jenkins folder
 - `mkdir -p /var/jenkins_home/.gradle`
 - `chown 1000:1000 /var/jenkins_home/.gradle`

Example:
```
node {
  def myGradleContainer = docker.image('gradle:jdk8-alpine')

  myGradleContainer.pull()
  stage('prep') {
    checkout scm
  }
  stage('test') {
     myGradleContainer.inside("-v ${env.HOME}/.gradle:/home/gradle/.gradle") {
       sh 'cd complete && gradle test'
     }
  }
  stage('run') {
     myGradleContainer.inside("-v ${env.HOME}/.gradle:/home/gradle/.gradle") {
       sh 'cd complete && gradle run'
     }
  }
}
```

#### Bitbucket Integration
- Install `Bitbucket Branch Source Plugin` from `Dashboad > Manage Jenkins > Manage Plugins`
- Go to `Manage Jenkins > Configure System` and you can configure your `Bitbucket Branch Source Plugin`
- Create a new item as `Bitbucket Team/Project`
- Under `Project`, add **credentials**.
 - To add credentials, go to `Bitbucket > Settings > App Passwords > Create App Password` 
 - Would recomment to add all of the **read permissions**
- Make sure you also add `Team ID` as **owner**
 - Under `Project`, there is `Bitbucket Team/Project`. Do not forget to add **the team**.
  - In Bitbucket; you can open project under team. So it goes like `<TEAM>/<PROJECT>/<REPOSITORY>`
- Since we are using Gradle as compiler, we are going to create a `.gradle` file inside our Jenkins folder
 - `mkdir -p /var/jenkins_home/.gradle`
 - `chown 1000:1000 /var/jenkins_home/.gradle`

 > There is a check box named `Auto-register webhooks`. If enabled, Bitbucket will send notifications to Jenkins for each commit. To do that, make sure your Jenkins URL under Manage Jenkins > Configure System.

### Custom API Integration
Example:
```
// example of custom API
import groovy.json.JsonSlurperClassic 

@NonCPS
def jsonParse(def json) {
    new groovy.json.JsonSlurperClassic().parseText(json)
}

def repo = "<BITBUCKET_LOGIN>/<BITBUCKET_REPO>"

def token = httpRequest authentication: 'bitbucket-oauth', contentType: 'APPLICATION_FORM', httpMode: 'POST', requestBody: 'grant_type=client_credentials', url: 'https://bitbucket.org/site/oauth2/access_token'
def accessToken = jsonParse(token.content).access_token
def pr = httpRequest customHeaders: [[maskValue: true, name: 'Authorization', value: 'Bearer ' + accessToken]], url: "https://api.bitbucket.org/2.0/repositories/${repo}/pullrequests"

for (p in jsonParse(pr.content).values) { 
    println(p.source.branch.name)
}
```
- Install `HTTP Request Plugin` from `Dashboad > Manage Jenkins > Manage Plugins`
- Create an `API Key`, we use **Bitbucket** for this example
 - Go to `Bitbucket > Settings > Access Management > OAuth`
 - Click on `Add Consumer`
 - Give a name
 - Put `Jenkins URL` as **Callback URL** and **URL**
 - Give **read permissions** for **Repositories** and **Pull Requests**
- Go to **Credentials** and add username and password from Bitbucket
- Create a **Pipeline** item
- Under `Pipeline`, choose `Pipeline script from SCM`
- Under `SCM`, choose `Git`
- Give the **Git Repository URL** as `Repository URL`
- Give the **Jenkins file path** as `Script Path` (from the root folder of the project)
- Go to `Dashboard > Manage Jenkins > In-Process Script Approval`
- **Approve** the API by clicking the approve button on top Save and build

### SonarQube Integration
- `mkdir -p /jenkins/docker-compose.yaml`
- `mkdir -p /opt/sonarqube/data/`
- `mkdir -p /opt/sonarqube/logs/`
- `mkdir -p /opt/sonarqube/extensions/`
Docker-Compose Example:
```
version: '3.8'
services:
  jenkins:
    image: jenkins/jenkins:lts
    ports:
      - "8080:8080"
      - "50000:50000"
    networks:
      - jenkins
    volumes:
      - ../var/jenkins_home:/var/jenkins_home
      - ../var/run/docker.sock:/var/run/docker.sock
  postgres:
    image: postgres:9.6
    networks:
      - jenkins
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonarpasswd
    volumes:
      - ../var/postgres-data:/var/lib/postgresql/data
  sonarqube:
    image: sonarqube:5.6.6
    ports:
      - "9000:9000"
      - "9092:9092"
      - "5432:5432"
    networks:
      - jenkins
    environment:
      SONARQUBE_JDBC_USERNAME: sonar
      SONARQUBE_JDBC_PASSWORD: sonarpasswd
      SONARQUBE_JDBC_URL: "jdbc:postgresql://postgres:5432/sonar"
    volumes:
      - ../opt/sonarqube/data:/opt/sonarqube/data
      - ../opt/sonarqube/logs:/opt/sonarqube/logs
      - ../opt/sonarqube/extensions:/opt/sonarqube/extensions
    depends_on: 
      - postgres

networks:
  jenkins:
```

- `cd jenkins && docker-compose up -d`
- Install `SonarQube Scanner for Jenkins` from `Dashboad > Manage Jenkins > Manage Plugins`
- Open `SonarQube`, login and **generate token**
 - username/password is admin/admin for SonarQube
 - Go to `Administrator > My Account > Security` and `Generate Tokens`
- Go to `Dashboad > Manage Jenkins > Global Tool Configurations`
- Add `SonarQube Scanner` as sonar
- Go to `Dashboard > Manage Jenkins > Credentials`
- Click on the **downward arrow** next to `(global)` proved by `System`
- Choose `Secret text`
- Enter the key you generated from SonarQube
- Create an item as pipeline
- Under Pipeline, select `Pipeline script from SCM`
- Choose `Git` as SCM
- Give the repository URL
- Give the Jenkins file as Script Path

Example Jenkins File:
```
node {
    def myGradleContainer = docker.image('gradle:jdk8-alpine')
    myGradleContainer.pull()

    stage('prep') {
        git url: '<REPOSITORY>'
    }

    stage('build') {
      myGradleContainer.inside("-v ${env.HOME}/.gradle:/home/gradle/.gradle") {
        sh 'cd complete && /opt/gradle/bin/gradle build'
      }
    }

    stage('sonar-scanner') {
      def sonarqubeScannerHome = tool name: 'sonar', type: 'hudson.plugins.sonar.SonarRunnerInstallation'

      withCredentials([string(credentialsId: 'sonar', variable: 'sonarLogin')]) {
        sh "${sonarqubeScannerHome}/bin/sonar-scanner -e -Dsonar.host.url=http://sonarqube:9000 -Dsonar.login=${sonarLogin} -Dsonar.projectName=gs-gradle -Dsonar.projectVersion=${env.BUILD_NUMBER} -Dsonar.projectKey=GS -Dsonar.sources=complete/src/main/ -Dsonar.tests=complete/src/test/ -Dsonar.language=java -Dsonar.java.binaries=."
      }
    }
}
```

> Since we are using Docker-Compose, we put the ***container names*** as IP Addresses.

## Advanced Jenkins Usage
### Jenkins Slaves
In production environments, you typically want to host the **Jenkins web UI on a small master node**, and have **one or more worker nodes** (Jenkins Slaves)

Typically one worker has one or more **build executors** (building slots)

If a Jenkins node has **2 executors**, only **2 builds** can run in **parallel**

#### ***Static*** or ***Manual*** scaling
You can have more workers during working hours or no workers outside working hours.

#### ***Dynamic*** workker scaling
You have plugins that can scale Jenkins slaves for you:
- **The Amazon EC2 Plugin**: If your build cluster gets overloaded, the plugin will start new slave nodes automatically and if the nodes are idle, they will automatically get killed
- **Docker Plugin**: Uses a docker host to spin up a slave container, run a Jenkins build in it and tear it down
- **Amazon ECS Plugin**: Same as Docker Plugin, but the host is now a Docker Orchestrator, the EC2 Container Engine, which can host the docker containers and scale out when necessary
- **DigitalOcean Plugin**: Dynamically privisions droplets to be used as Jenkins Slaves

Builds can be executed on specific nodes. As nodes can be labeled.
Example:
```
node(label: '<NODE_LABEL>') {
    stage('build') {
        [...]
    }
}
```

###

## End


















