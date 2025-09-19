pipeline {
    agent any

    environment {
        GIT_URL = 'https://github.com/yunseo0611/myfirst-api-server1.git'
        GIT_BRANCH = 'main' // 또는 main
        GIT_ID = 'skala-github-id' // GitHub PAT credential ID
        IMAGE_REGISTRY = 'amdp-registry.skala-ai.com/skala25a'
        IMAGE_NAME = 'sk117-myfirst-api-server1'
        IMAGE_TAG = '1.0.0'
        DOCKER_CREDENTIAL_ID = 'skala-image-registry-id'  // Harbor 인증 정보 ID
        K8S_NAMESPACE = 'skala-practice'
    }

    options {
        disableConcurrentBuilds()   // 같은 Job 동시 실행 금지
        timestamps()                // 콘솔 로그에 타임스탬프 표시
    }

    stages {
        stage('Clone Repository') {
            steps {
                git branch: "${GIT_BRANCH}",
                    url: "${GIT_URL}",
                    credentialsId: "${GIT_ID}"   // GitHub PAT credential ID
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    // 해시코드 12자리 생성
                    def hashcode = sh(
                        script: "date +%s%N | sha256sum | cut -c1-12",
                        returnStdout: true
                    ).trim()

                    def FINAL_IMAGE_TAG = "${IMAGE_TAG}-${hashcode}"
                    echo "Final Image Tag: ${FINAL_IMAGE_TAG}"

                    docker.withRegistry("https://${IMAGE_REGISTRY}", "${DOCKER_CREDENTIAL_ID}") {
                        def appImage = docker.build("${IMAGE_REGISTRY}/${IMAGE_NAME}:${FINAL_IMAGE_TAG}", "--platform=linux/amd64 .")
                        appImage.push()
                    }

                    // 최종 이미지 태그를 env에 등록 (나중에 deploy.yaml 수정에 사용)
                    env.FINAL_IMAGE_TAG = FINAL_IMAGE_TAG
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    # 1) image 경로까지만 그룹(\\1)으로 잡고, 콜론은 그룹 밖으로 뺌
                    #    (image:[공백]*REG/NAME)\\S+  ⇒ 기존 태그 전체를 치환
                    sed -Ei "s#(image:[[:space:]]*${IMAGE_REGISTRY}/${IMAGE_NAME})[^[:space:]]+#\\1:${FINAL_IMAGE_TAG}#" ./k8s/deploy.yaml
                    
                    echo '=== image after replace ==='
                    grep -n 'image:' ./k8s/deploy.yaml

                    kubectl apply -n ${K8S_NAMESPACE} -f ./k8s
                    kubectl rollout status -n ${K8S_NAMESPACE} deployment/${IMAGE_NAME}
                """
            }
        }
    }
}


