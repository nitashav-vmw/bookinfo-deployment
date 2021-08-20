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
    - name: IMGPKG_REGISTRY_HOSTNAME_1
      value: "carvel-harbor.customer0.io"
    - name: IMGPKG_REGISTRY_USERNAME_1
      valueFrom:
        secretKeyRef:
          name: bookinfo-secrets
          key: registry-user
    - name: IMGPKG_REGISTRY_PASSWORD_1
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
        stage('checkout'){
            environment { 
                GIT_AUTH = credentials('f09786ed-8c24-4d1a-a768-f0b5266383be') 
            }
            steps{
                container('jnlp'){
                    sh('''
                        git config --global credential.helper "!f() { echo username=\\$GIT_AUTH_USR; echo password=\\$GIT_AUTH_PSW; }; f"
                        git config --global user.email "nitashav@vmware.com"
                        git config --global user.name "nitashav-vmw"
                        git clone https://github.com/nitashav-vmw/bookinfo-deployment.git
                    ''')
                }
            }
        }
        
        stage('bundle-pull') {
            steps {
              sh('''
              ls .
              echo $data
              mkdir local
              imgpkg copy -b harbor.tools.azure.nvcodes.net/isv-release/bookinfo:$data --to-repo carvel-harbor.customer0.io/bookinfo-bundle/dependencies 
              imgpkg pull -b carvel-harbor.customer0.io/bookinfo-bundle/dependencies:$data   -o ./local
              ls ./local
              ''') 
             
            }
        }
        stage('esf-container-relocate'){
            steps{
                sh ('''
                cd local
                FILES=./resource-definitions/**/*
                for f in $FILES
                do
                  echo "$f"
                  nf="$(basename $f)"
                  echo "Processing $nf file..."
                  # take action on each file. $nf store current file name
                  kbld -f $f -f ./.imgpkg/images.yml > $f.resolved
                  mv $f.resolved $f
                done
               
                cd ../bookinfo-deployment
                ##mkdir ./base
                 cp ../local/resource-definitions/**/* ./base/
                 ls ./base
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
                        cd ./bookinfo-deployment
                        git config --local credential.helper "!f() { echo username=\\$GIT_AUTH_USR; echo password=\\$GIT_AUTH_PSW; }; f"
                        git config --global user.email "nitashav@vmware.com"
                        git config --global user.name "nitashav-vmw"
                        #git fetch origin main
                        #git merge origin/main
                        git branch -a
                        git add ./base/*
                        git diff-index --quiet main ||git commit  -m "Latest changes from ISV"
                        git branch -u origin/main
                        ##git branch --set-upstream-to origin/main
                        git push origin main
                    ''')
                }
            }
        }
    }
}
