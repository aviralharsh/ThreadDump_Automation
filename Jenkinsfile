def env='preprod'

pipeline {
  agent {
        kubernetes {
            label 'jenkins-slave-threadDump'
            defaultContainer 'jnlp'
            yamlFile 'KubernetesPod.yaml'
        }
    }
  
  environment{
  	bucket = 'blibli-thread-dump'
  }

  stages {

    stage('Connect to cluster'){
      steps {
        container('kubectl'){
          withCredentials([file(credentialsId: 'gke-qa1-cluster-serviceaccount', variable: 'SERVICE_ACCOUNT')]) {
            sh("gcloud auth activate-service-account --key-file=${SERVICE_ACCOUNT}")
                  }
            sh ("gcloud container clusters get-credentials gke-preprod-sg --region asia-southeast1 --project gke-preprod --internal-ip")
        }
      }
    }

   stage('Create Thread Dump'){
     steps {
       script {
         container('kubectl'){
             def podDetail = readYaml(file: 'Pod_Name.yaml') ['podDetails']
             for (pod in podDetail) {
               def pid = sh(script:"kubectl exec -i ${pod['name']} -n ${pod['namespace']} -- /bin/sh -c 'pgrep -f java'", returnStdout: true).trim()
               if (pid == 1) {
                 echo "Please use new docker image"
               }
           
               else {
                 // Taking thread dump
                 sh ("kubectl exec -i ${pod['name']} -n ${pod['namespace']} -- /bin/sh -c 'jstack -l ${pid} > /tmp/threadDump.txt'")
                 sh ("kubectl cp ${pod['namespace']}/${pod['name']}:/tmp/threadDump.txt ./${pod['name']}_threadDump.txt")
             
                 // Uploading to GCP Bucket
                 sh("gsutil cp ./${pod['name']}_threadDump.txt gs://${bucket}/${env}/${pod['name']}_threadDump.txt")
             
                 // POD Cleanup
                 sh ("kubectl exec -i ${pod['name']} -n ${pod['namespace']} -- /bin/sh -c 'rm /tmp/threadDump.txt'")
                 
                 echo "Thread Dump extraction complete." 
                 echo "You can find the dump at: https://console.cloud.google.com/storage/browser/${bucket}/${env};tab=objects?forceOnBucketsSortingFiltering=false&project=nonprod-utility-233414"
              }
            }
          }
        }
      }
    }
  }
}