pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
metadata:
  labels:
    jenkins/label: java-k8s
spec:
  serviceAccountName: jenkins-sa
  containers:
    - name: jnlp
      image: jenkins/inbound-agent:latest
      args: ['$(JENKINS_SECRET)', '$(JENKINS_NAME)']
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: maven
      image: maven:3.9.4-jdk-17
      command: ['sh', '-c', 'sleep infinity']
      tty: true
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: trufflehog
      image: python:3.12-alpine
      command: ['sh', '-c', 'sleep infinity']
      tty: true
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: owasp
      image: owasp/dependency-check:latest
      command: ['sh', '-c', 'sleep infinity']
      tty: true
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

    - name: kaniko
      image: gcr.io/kaniko-project/executor:debug
      command: ['sh', '-c', 'sleep infinity']
      tty: true
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent
        - name: harbor-creds
          mountPath: /kaniko/.docker/config.json
          subPath: config.json

    - name: kubectl
      image: lachlanevenson/k8s-kubectl:latest
      command: ['sh', '-c', 'sleep infinity']
      tty: true
      workingDir: /home/jenkins/agent
      volumeMounts:
        - name: workspace-volume
          mountPath: /home/jenkins/agent

  volumes:
    - name: harbor-creds
      secret:
        secretName: harbor-creds
        items:
          - key: .dockerconfigjson
            path: config.json
    - name: workspace-volume
      emptyDir: {}
'''
    }
  }

  environment {
    APP_NAME      = "dptweb"
    GIT_REPO      = "https://github.com/sebastined/ann-java-login.git"
    REGISTRY      = "harbor.int.sebastine.ng/900"
    IMAGE_NAME    = "${REGISTRY}/${APP_NAME}"
    TAG           = "${BUILD_NUMBER}"
    K8S_NAMESPACE = "dev00"
    INGRESS_HOST  = "${APP_NAME}.int.seba
