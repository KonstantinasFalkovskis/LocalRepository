# Local images repository at HIPERION
http://www.vinsguru.com/selenium-docker-integration-through-jenkinsfile-managing-local-docker-registry/
http://www.testautomationguru.com/selenium-docker-integration-through-jenkinsfile-part-3-executing-tests-inside-docker-container/

NOTE: For starting docker local registry
https://docs.docker.com/registry/
1. docker run -d -p 5000:5000 --name registry -v /mnt/volume/images:/var/lib/registry registry:2
2. docker image tag ubuntu 192.168.0.106:5000/myfirstimage
3. docker push 192.168.0.106:5000/myfirstimage
4. docker pull 192.168.0.106:5000/myfirstimage
5. docker container stop registry && docker container rm -v registry

1. Start local images repository on the server in deamon mode:
   - sudo docker-compose up -d

2. reach the registry locally from the browser	
   - http://localhost 

3. Modifyi or create daemon.json

path: /etc/docker/daemon.json
cd /etc/docker/
ls -al
sudo nano daemon.json

update existing file:

{
   "insecure-registries":[
      "10.11.12.13:5000"
   ]
}

4. restart docker
- systemctl restart docker
- systemctl status docker

Now each machine in the network is ready to use our Docker-registry.

5. Push to the repository
- docker push 192.168.0.106:5000/hiperion/tests

6. Pull from the repository
- docker pull 192.168.0.106:5000/hiperion/tests

7. Pushing to Docker Repository with Jenkins Pepiline
pipeline {
    agent {
        node {
            label 'docker' && 'maven'
        }
    }
    stages {    
        stage('Build Jar') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }
        stage('Build Image') {
            steps {
                      app = docker.build("vinsdocker/containertest")
                }
            }
        }
        stage('Push Image') {
            steps {
                script {
                    // http://10.11.12.13:5000 is our docker registry
                    docker.withRegistry('http://10.11.12.13:5000') {
                        app.push("${BUILD_NUMBER}")
                        app.push("latest")
                    }
                }
            }
        }        
    }
}

8. Running Our Selenium Tests Using Jenkins:
pipeline {
  
   agent { label 'docker' }

   stages{
      stage('Setting Up Selenium Grid') {
         steps{        
            sh "docker network create ${network}"
            sh "docker run -d -p 4444:4444 --name ${seleniumHub} --network ${network} selenium/hub"
            sh "docker run -d -e HUB_PORT_4444_TCP_ADDR=${seleniumHub} -e HUB_PORT_4444_TCP_PORT=4444 --network ${network} --name ${chrome} selenium/node-chrome"
            sh "docker run -d -e HUB_PORT_4444_TCP_ADDR=${seleniumHub} -e HUB_PORT_4444_TCP_PORT=4444 --network ${network} --name ${firefox} selenium/node-firefox"
         }
      }
      stage('Run Test') {
         steps{
            parallel(
               "search-module":{
                  sh "docker run --rm -e SELENIUM_HUB=${seleniumHub} -e BROWSER=firefox -e MODULE=search-module.xml -v ${WORKSPACE}/search:/usr/share/tag/test-output --network ${network} 10.11.12.13:500/vinsdocker/containertest"
                  archiveArtifacts artifacts: 'search/**', fingerprint: true
               },
               "order-module":{
                  sh "docker run --rm -e SELENIUM_HUB=${seleniumHub} -e BROWSER=chrome -e MODULE=order-module.xml -v ${WORKSPACE}/order:/usr/share/tag/test-output  --network ${network} 10.11.12.13:500/vinsdocker/containertest"
                  archiveArtifacts artifacts: 'order/**', fingerprint: true
               }               
            ) 
         }
      }
    }
    post{
      always {
         sh "docker rm -vf ${chrome}"
         sh "docker rm -vf ${firefox}"
         sh "docker rm -vf ${seleniumHub}"
         sh "docker network rm ${network}"
      }   
   }
}
