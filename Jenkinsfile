// Uses Declarative syntax to run commands inside a container.
pipeline {
    agent {
        kubernetes {
            // Rather than inline YAML, in a multibranch Pipeline you could use: yamlFile 'jenkins-pod.yaml'
            // Or, to avoid YAML:
            // containerTemplate {
            //     name 'shell'
            //     image 'ubuntu'
            //     command 'sleep'
            //     args 'infinity'
            // }
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: shell
    image: taylorsilva/carvel-apps
    env:
    - name: KBLD_REGISTRY_HOSTNAME_0
      value: "harbor.tools.azure.nvcodes.net"
    - name: KBLD_REGISTRY_USERNAME_0
      valueFrom:
        secretKeyRef:
          name: bookinfo-secrets
          key: registry-user
    - name: KBLD_REGISTRY_PASSWORD_0
      valueFrom:
        secretKeyRef:
          name: bookinfo-secrets
          key: registry-password
    - name: IMGPKG_REGISTRY_HOSTNAME_0
      value: "harbor.tools.azure.nvcodes.net"
    - name: IMGPKG_REGISTRY_USERNAME_0
      valueFrom:
        secretKeyRef:
          name: bookinfo-secrets
          key: registry-user
    - name: IMGPKG_REGISTRY_PASSWORD_0
      valueFrom:
        secretKeyRef:
          name: bookinfo-secrets
          key: registry-password
    command:
    - sleep
    args:
    - infinity
'''
            // Can also wrap individual steps:
            // container('shell') {
            //     sh 'hostname'
            // }
            defaultContainer 'shell'
        }
    }
    stages {
        
        stage('bundle-pull') {
            steps {
              sh('''
              echo $data
              mkdir local
              imgpkg copy -b harbor.tools.azure.nvcodes.net/isv-release/bookinfo:$data --to-repo harbor.tools.azure.nvcodes.net/bookinfo-bundle/dependencies 
              imgpkg pull -b harbor.tools.azure.nvcodes.net/bookinfo-bundle/dependencies:$data   -o ./local
              ls . 
              ''') 
             
            }
        }
        stage('esf-container-relocate'){
            steps{
                sh ('''
                cd local
                FILES=./resource-definitions/**/*
                mkdir new-deployment
                for f in $FILES
                do
                  echo "$f"
                  nf="$(basename $f)"
                  echo "Processing $nf file..."
                  # take action on each file. $nf store current file name
                  kbld -f ./deployment/$nf -f ./.imgpkg/images.yml > ./new-deployment/$nf
                done
                
                mv ./new-deployment/* ./deployment/
                
                rm -r ./new-deployment
                
                cd ..
                
                cp ./local/deployment/* ./esf-gitops/base/
                ''')
            }
        }
        stage('Push') {
            environment { 
                GIT_AUTH = credentials('f09786ed-8c24-4d1a-a768-f0b5266383be') 
            }
            steps {
                container('jnlp'){
                    sh('''
                        cd esf-gitops/base
                        git config --local credential.helper "!f() { echo username=\\$GIT_AUTH_USR; echo password=\\$GIT_AUTH_PSW; }; f"
                        git config --global user.email "nitashav@vmware.com"
                        git config --global user.name "nitashav-vmw"
                        
                        git add .
                        git diff-index --quiet main ||git commit  -m "Latest changes from Fiserv"
                        git branch -u origin/main
                        git push origin main
                    ''')
                }
            }
        }
    }
}
