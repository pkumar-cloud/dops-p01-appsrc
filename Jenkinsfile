def imageName = 'stalinrtp.jfrog.io/valaxy-docker/valaxy-rtp'
def registry  = 'https://stalinrtp.jfrog.io'
def version   = '1.0.0' // Docker image vrsion
def app
pipeline {
    agent {
       node {
         label "worker"
      }
    }
    stages {
        stage('Build') {
            steps {
                echo '<--------------- Building --------------->'
                sh 'printenv'
                sh 'mvn clean deploy -Dmaven.test.skip=true'
                echo '<------------- Build completed --------------->'
            }
        }
        stage('Unit Test') {
            steps {
                echo '<--------------- Unit Testing started  --------------->'
                sh 'mvn surefire-report:report'
                echo '<------------- Unit Testing stopped  --------------->'
            }
        }
        
        stage ("Sonar Analysis") {
            environment {
               scannerHome = tool 'SonarQubeScanner' // Name defined under Manage Jenkins => Global Tool Config => SonarQubeScanner => Name
            }
            steps {
                echo '<--------------- Sonar Analysis started  --------------->'
                withSonarQubeEnv('SonarServer') { // Name defined under Manage Jenkins => Configure System => SonarQube installations => Name
                    sh "${scannerHome}/bin/sonar-scanner" // Will look for details in sonar-project.properties, scan and push the results to server.
                }    
                echo '<--------------- Sonar Analysis stopped  --------------->'
            }   
        }    
        
          stage ("Quality Gate") {

            steps {
                script {
                  echo '<--------------- Quality Gate started  --------------->' 
                    timeout(time: 1, unit: 'HOURS') {  // Timeout afetr 1 hour.
                        def qg = waitForQualityGate() // Jenkins will keeo waiting
                        if(qg.status!='OK'){
                          error "Pipeline failed due to the Quality gate issue"   
                        }    
                    }    
                  echo '<--------------- Quality Gate stopped  --------------->'
                }    
            }   
        }          
        
        stage(" Docker Build ") {
          steps {
            script {
               echo '<--------------- Docker Build Started --------------->'
               app = docker.build(imageName+":"+version)
               echo '<--------------- Docker Build Ends --------------->'
            }
          }
        }
        
        stage("Jar Publish") {
            steps {
                script {
                        echo '<--------------- Jar Publish Started --------------->'
                         def server = Artifactory.newServer url:registry+"/artifactory" ,  credentialsId:"artifactorycredentialid" // Credential should be same as defined in jenkins global credentials.
                         def properties = "buildid=${env.BUILD_ID},commitid=${GIT_COMMIT}";
                         // target: Name of Jfrog virtual repo, defines below what data to upload, 
                         def uploadSpec = """{ 
                              "files": [
                                {
                                  "pattern": "jarstaging/(*)",
                                  "target": "valaxy-libs-release/{1}", 
                                  "flat": "false",
                                  "props" : "${properties}",
                                  "exclusions": [ "*.sha1", "*.md5"]
                                }
                             ]
                         }"""
                         def buildInfo = server.upload(uploadSpec) 
                         buildInfo.env.collect()
                         server.publishBuildInfo(buildInfo)
                         echo '<--------------- Jar Publish Ended --------------->'  
                
                }
            }   
        }    
        
        stage (" Docker Publish "){
            steps {
                script {
                   echo '<--------------- Docker Publish Started --------------->'  
                    docker.withRegistry(registry, 'dockercredentialid'){ // Credential should be same as defined in jenkins global credentials. 
                        app.push()
                        // docker.image(imageName).push(version)
                    }    
                   echo '<--------------- Docker Publish Ended --------------->'  
                }
            }
        }
        
         stage(" Deploy ") {
          steps {
            script {
               input: "Proceed or not ?" // Ask for manual confirmation
               echo '<--------------- Deploy Started --------------->'
               sh './deploy.sh'
               echo '<--------------- Deploy Ends --------------->'
            }
          }
        }    
    }
 }
