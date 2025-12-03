//---------------------------------------------
// Author: Adam WezvaTechnologies
// Call/Whatsapp: +91-9739110917
//---------------------------------------------

pipeline {
agent {label 'demo'}

stages {
  stage('Install Helm'){
    steps {
            sh 'curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash'
    }
  }
  stage('Scan Manifests'){
    steps {
      catchError(buildResult: 'SUCCESS', message: 'IAC Misconfigurations found', stageResult: 'UNSTABLE')
      {
             sh 'helm template ./mychart -f envs/functional/values.yaml > renderedmanifest.yaml'
             sh 'docker run --rm -v "$(pwd):/app" kubesec/kubesec scan /app/renderedmanifest.yaml'
       }
    }
  }
}
}

//---------------------------------------------
// Author: Adam WezvaTechnologies
// Call/Whatsapp: +91-9739110917
//---------------------------------------------