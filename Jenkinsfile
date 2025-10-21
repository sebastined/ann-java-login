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
      image: maven
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
    INGRESS_HOST  = "${APP_NAME}.int.sebastine.ng"
  }

  stages {
    stage('Prepare workspace') {
      steps {
        container('jnlp') {
          sh 'git config --global --add safe.directory /home/jenkins/agent || true'
        }
      }
    }

    stage('Checkout Source') {
      steps {
        container('jnlp') {
          git branch: 'master', url: "${GIT_REPO}"
        }
      }
    }

    stage('Initialize Environment') {
      steps {
        container('maven') {
          sh '''
            echo "PATH = ${PATH}"
            echo "M2_HOME = ${M2_HOME}"
          '''
        }
      }
    }

    stage('Check Git Secrets') {
      steps {
        container('trufflehog') {
          sh '''
            pip install trufflehog
            rm -f trufflehog.json || true
            trufflehog --json ${GIT_REPO} > trufflehog.json
            cat trufflehog.json
          '''
        }
      }
    }

    stage('Source Composition Analysis') {
      steps {
        container('owasp') {
          sh '''
            rm -f owasp* || true
            wget "https://raw.githubusercontent.com/sebastined/ann-java-login/master/owasp-dependency-check.sh"
            chmod +x owasp-dependency-check.sh
            bash owasp-dependency-check.sh
            cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml
          '''
        }
      }
    }

    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn clean package'
        }
      }
    }

    stage('Build & Push Docker Image') {
      steps {
        container('kaniko') {
          sh '''
            IMAGE_DEST="${IMAGE_NAME}:${TAG}"
            echo "Building ${IMAGE_DEST} with Kaniko..."
            /kaniko/executor \
              --context "$PWD" \
              --dockerfile Dockerfile \
              --destination "${IMAGE_DEST}" \
              --cache=true \
              --insecure --skip-tls-verify
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            echo "Deploying ${APP_NAME}:${TAG} to ${K8S_NAMESPACE}..."

            # Deployment
            kubectl apply -n ${K8S_NAMESPACE} -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}-deploy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      imagePullSecrets:
        - name: harbor-creds
      containers:
      - name: ${APP_NAME}-container
        image: ${IMAGE_NAME}:${TAG}
        ports:
        - containerPort: 8080
EOF

            # Service
            kubectl apply -n ${K8S_NAMESPACE} -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}-svc
spec:
  selector:
    app: ${APP_NAME}
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP
EOF

            # Ingress (Traefik)
            kubectl apply -n ${K8S_NAMESPACE} -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${APP_NAME}-ing
  annotations:
    kubernetes.io/ingress.class: traefik
    traefik.ingress.kubernetes.io/router.entrypoints: web,websecure
spec:
  ingressClassName: traefik
  tls:
  - hosts:
      - ${INGRESS_HOST}
    secretName: int-wildcard
  rules:
  - host: ${INGRESS_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${APP_NAME}-svc
            port:
              number: 8080
EOF

            # Wait for rollout
            kubectl -n ${K8S_NAMESPACE} rollout status deployment/${APP_NAME}-deploy --timeout=120s
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Successfully built and deployed ${APP_NAME}:${TAG} to Kubernetes"
    }
    failure {
      echo "❌ Pipeline failed — check logs for details"
    }
  }
}
