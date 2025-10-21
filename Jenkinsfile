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
      image: maven:3.9.4-openjdk-17
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
    TRUFFLE_NAME  = "trufflehog"
  }

  stages {
    stage('Prepare Workspace') {
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

    stage('Build Java App') {
      steps {
        container('maven') {
          sh 'mvn -B -DskipTests clean package'
        }
      }
    }

    stage('Build & Push Java Image') {
      steps {
        container('kaniko') {
          sh '''
            set -e
            IMAGE_DEST="${IMAGE_NAME}:${TAG}"
            echo "Building and pushing Java image: ${IMAGE_DEST}"
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

    stage('Build & Push TruffleHog Image (tagged)') {
      steps {
        container('kaniko') {
          sh '''
            set -e
            IMAGE_TRUFFLE="${REGISTRY}/${TRUFFLE_NAME}:${TAG}"
            echo "Building and pushing TruffleHog image: ${IMAGE_TRUFFLE}"
            /kaniko/executor \
              --context "$PWD" \
              --dockerfile T-Dockerfile \
              --destination "${IMAGE_TRUFFLE}" \
              --cache=true \
              --insecure --skip-tls-verify
          '''
        }
      }
    }

    stage('Run TruffleHog Scan (transient pod)') {
      steps {
        container('kubectl') {
          sh '''
            set -e
            POD_NAME="trufflehog-scan-${TAG}"
            IMAGE_TRUFFLE="${REGISTRY}/${TRUFFLE_NAME}:${TAG}"
            echo "Launching transient pod ${POD_NAME} in namespace ${K8S_NAMESPACE} with image ${IMAGE_TRUFFLE}..."

            kubectl apply -n ${K8S_NAMESPACE} -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ${POD_NAME}
  labels:
    app: trufflehog-scan
spec:
  restartPolicy: Never
  imagePullSecrets:
    - name: harbor-creds
  containers:
    - name: trufflehog
      image: ${IMAGE_TRUFFLE}
      # If the image ENTRYPOINT is trufflehog you may not need command/args;
      # using args here to be explicit
      command: ["trufflehog"]
      args: ["--json", "${GIT_REPO}"]
EOF

            # wait for pod to start and finish
            echo "Waiting for pod to enter a terminal phase..."
            for i in $(seq 1 60); do
              PHASE=$(kubectl -n ${K8S_NAMESPACE} get pod ${POD_NAME} -o jsonpath='{.status.phase}' 2>/dev/null || echo "NotFound")
              echo "  ${POD_NAME} phase: ${PHASE}"
              if [ "${PHASE}" = "Succeeded" ] || [ "${PHASE}" = "Failed" ]; then
                break
              fi
              sleep 2
            done

            echo "Collecting logs to trufflehog-${TAG}.json"
            kubectl -n ${K8S_NAMESPACE} logs ${POD_NAME} > trufflehog-${TAG}.json || true
            echo "----- TruffleHog output (first 200 lines) -----"
            sed -n '1,200p' trufflehog-${TAG}.json || true

            # cleanup pod
            kubectl -n ${K8S_NAMESPACE} delete pod ${POD_NAME} --ignore-not-found --wait=true || true
          '''
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        container('kubectl') {
          sh '''
            set -e
            echo "Deploying ${APP_NAME}:${TAG} to ${K8S_NAMESPACE}..."

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

            kubectl -n ${K8S_NAMESPACE} rollout status deployment/${APP_NAME}-deploy --timeout=120s
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Success: built images (tag ${TAG}), ran TruffleHog, and deployed ${APP_NAME}:${TAG}"
      archiveArtifacts artifacts: "trufflehog-${TAG}.json", allowEmptyArchive: true
    }
    failure {
      echo "❌ Pipeline failed — check logs"
      archiveArtifacts artifacts: "trufflehog-${TAG}.json", allowEmptyArchive: true
    }
  }
}
