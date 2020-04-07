def label = "worker-${UUID.randomUUID().toString()}"

podTemplate(label: label, containers: [
  containerTemplate(name: 'docker', image: 'docker:18.09', command: 'cat', ttyEnabled: true),
  containerTemplate(name: 'argo-cd', image: 'argoproj/argo-cd-ci-builder:v1.0.0', command: 'cat', ttyEnabled: true)
],
volumes: [
  hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock')
]) {
  node(label) {
    def myRepo = checkout scm
    def gitCommit = myRepo.GIT_COMMIT
    def gitBranch = myRepo.GIT_BRANCH
    def shortGitCommit = "${gitCommit[0..10]}"
    def previousGitCommit = sh(script: "git rev-parse ${gitCommit}~", returnStdout: true)


    stage('Create Docker images') {
      container('docker') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
          credentialsId: 'dockerhub',
          usernameVariable: 'DOCKER_HUB_USER',
          passwordVariable: 'DOCKER_HUB_PASSWORD']]) {
          sh """
            docker login -u ${DOCKER_HUB_USER} -p ${DOCKER_HUB_PASSWORD}
            docker build -t ${DOCKER_HUB_USER}/rsvp-demo:${gitCommit} .
            docker push ${DOCKER_HUB_USER}/rsvp-demo:${gitCommit}
            """
        }
      }
    }
    stage('Deploy to Staging') {
      container('argo-cd') {
        withCredentials([[$class: 'UsernamePasswordMultiBinding',
          credentialsId: 'githubcred',
          usernameVariable: 'GITHUB_USER',
          passwordVariable: 'GITHUB_PASSWORD']]) {
          sh """
            KUSTOMIZE_REPO=github.com/nkhare/rsvpapp-kustomize
            USER_EMAIL=neependra.khare@gmail.com
            git clone https://$GITHUB_USER:$GITHUB_PASSWORD@$KUSTOMIZE_REPO
            git config --global user.email $USER_EMAIL
            cd rsvpapp-kustomize/overlays/staging && kustomize edit set image ${env.IMAGE_REPO}:${env.GIT_COMMIT}
            git add . 
            git commit -am 'Publish new version' && git push || echo 'no changes'
             """
        }
      }
    }
  }
}
