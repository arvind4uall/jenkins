pipeline{
    agent any
    stages{
        stage("SCM"){
            steps{
                echo "Hello from SCM"
            }
        }
        stage("Docker Build"){
            steps{
                echo "Hello from Build"
            }
        }
        stage("Test"){
                parallel{
                    stage("security testing"){
                    steps{
                        echo "Testing CVE"
                        sh "sleep 100"
                    } 
                }
                stage("Integration testing"){
                    steps{
                        echo "Integration testing is going on..."
                        sh "sleep 100"
                    } 
                }
                stage("Karate testing"){
                    steps{
                        echo "Karate testing is going on"
                        sh "sleep 100"
                    } 
                }
                }
            
        }
        stage('Deploy approval'){
            steps{
                input "Deploy to prod?"
            }
            
            }
        stage("Prod deploy"){
            steps{
                echo "Hello from prod deploy"
            }
        }
    }
}
