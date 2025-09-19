pipeline {
    agent {
        kubernetes {
            cloud 'skala'  // 생성한 Cloud 이름 지정
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  # Git 및 Maven 작업용 컨테이너
  - name: maven
    image: maven:3.8.5-openjdk-17
    command:
    - cat
    tty: true
    volumeMounts:
    - name: maven-cache
      mountPath: /root/.m2
  # Kaniko 컨테이너 (Docker 빌드/푸시용)
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/cat
    tty: true
    env:
    - name: DOCKER_CONFIG
      value: /kaniko/.docker
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
      readOnly: true
  # Git 작업용 컨테이너
  - name: git
    image: alpine/git:latest
    command:
    - cat
    tty: true
  volumes:
  - name: docker-config
    secret:
      secretName: docker-registry-secret
      items:
      - key: .dockerconfigjson
        path: config.json
  - name: maven-cache
    emptyDir: {}
  restartPolicy: Never
"""
        }
    }
    environment {
        GIT_URL = 'https://github.com/yunseo0611/myfirst-api-server1.git'
        GIT_BRANCH = 'main' // 또는 main
        GIT_ID = 'skala-github-id' // GitHub PAT credential ID
        GIT_USER_NAME = 'yunseo0611' // GitHub 사용자 이름
        GIT_USER_EMAIL = 'joyunseo0611@gmail.com'
        IMAGE_REGISTRY = 'amdp-registry.skala-ai.com/skala25a'
        IMAGE_NAME = 'sk117-myfirst-api-server1'
        IMAGE_TAG = '1.0.0'
        DOCKER_CREDENTIAL_ID = 'skala-image-registry-id' // Harbor 인증 정보 ID
        K8S_NAMESPACE = 'skala-practice'
    }
    options {
        disableConcurrentBuilds() // 같은 Job 동시 실행 금지
        timestamps() // 콘솔 로그에 타임스탬프 표시
    }
    stages {
        stage('Clone Repository') {
            steps {
                container('git') {
                    git branch: "${GIT_BRANCH}",
                        url: "${GIT_URL}",
                        credentialsId: "${GIT_ID}" // GitHub PAT credential ID
                }
            }
        }
        
        stage('Build with Maven') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }
        
        stage('Docker Build & Push') {
            steps {
                container('kaniko') {
                    script {
                        // 해시코드 12자리 생성
                        def hashcode = sh(
                            script: "date +%s%N | sha256sum | cut -c1-12",
                            returnStdout: true
                        ).trim()
                        def FINAL_IMAGE_TAG = "${IMAGE_TAG}-${hashcode}"
                        echo "Final Image Tag: ${FINAL_IMAGE_TAG}"
                        
                        // kaniko로 이미지 빌드 및 푸시 (기존 docker 명령어 대체)
                        sh """
                            /kaniko/executor \\
                                --dockerfile=Dockerfile \\
                                --context=. \\
                                --destination=${IMAGE_REGISTRY}/${IMAGE_NAME}:${FINAL_IMAGE_TAG} \\
                                --force
                        """
                        
                        // 최종 이미지 태그를 env에 등록 (나중에 deploy.yaml 수정에 사용)
                        env.FINAL_IMAGE_TAG = FINAL_IMAGE_TAG
                    }
                }
            }
        }

        stage('Update deploy.yaml and Git Push') {
            steps {
                container('git') {
                    script {
                        // 환경변수 값 확인
                        echo "=== Environment Variables Check ==="
                        echo "GIT_ID: ${env.GIT_ID}"
                        echo "FINAL_IMAGE_TAG: ${env.FINAL_IMAGE_TAG}"
                        echo "GIT_BRANCH: ${env.GIT_BRANCH}"
                        echo "GIT_URL: ${env.GIT_URL}"
                        
                        def gitRepoPath = env.GIT_URL.replaceFirst(/^https?:\/\//, '')
                        echo "gitRepoPath: ${gitRepoPath}"
                        
                        sh """
                            set -euxo pipefail  # 디버깅 모드 활성화
                            
                            # 필요한 도구 설치 (alpine 기반)
                            apk add --no-cache sed grep
                            
                            # 현재 상태 확인
                            echo "=== 현재 디렉토리와 파일 ==="
                            pwd
                            ls -la
                            
                            # deploy.yaml 파일 존재 확인
                            if [ -f "./k8s/deploy.yaml" ]; then
                                echo "deploy.yaml found"
                                echo "=== 수정 전 deploy.yaml ==="
                                cat ./k8s/deploy.yaml
                            else
                                echo "ERROR: deploy.yaml not found"
                                find . -name "*.yaml" -o -name "*.yml" | head -10
                                exit 1
                            fi
                            
                            # 1) image 경로까지만 그룹(\\1)으로 잡고, 콜론은 그룹 밖으로 뺌
                            # (image:[공백]*REG/NAME)\\S+ ⇒ 기존 태그 전체를 치환
                            sed -Ei "s#(image:[[:space:]]*${IMAGE_REGISTRY}/${IMAGE_NAME})[^[:space:]]+#\\1:${FINAL_IMAGE_TAG}#" ./k8s/deploy.yaml

                            echo "=== 수정 후 deploy.yaml ==="
                            cat ./k8s/deploy.yaml
                            echo '=== image lines after replace ==='
                            grep -n 'image:' ./k8s/deploy.yaml
  
                            # 2) git config 설정
                            echo "=== Git 설정 ==="
                            git config --global --add safe.directory '*'
                            git config --global user.name "$GIT_USER_NAME"
                            git config --global user.email "$GIT_USER_EMAIL"
                            echo "Git user configured: \$(git config user.name) <\$(git config user.email)>"
                            
                            # 3) gitops 브랜치 확인 및 생성/체크아웃
                            echo "=== GitOps 브랜치 처리 ==="
                            # 원격 브랜치 정보 가져오기
                            git fetch origin || true
                            
                            # 현재 master 브랜치의 deploy.yaml 백업
                            cp ./k8s/deploy.yaml ./k8s/deploy.yaml.backup
                            
                            # gitops 브랜치가 존재하는지 확인 (로컬 또는 원격)
                            if git show-ref --verify --quiet refs/heads/gitops || git show-ref --verify --quiet refs/remotes/origin/gitops; then
                                echo "gitops 브랜치가 존재합니다. 강제 체크아웃합니다."
                                git checkout -f gitops || git checkout -B gitops origin/gitops
                            else
                                echo "gitops 브랜치가 존재하지 않습니다. 새로 생성합니다."
                                git checkout -b gitops
                            fi
                            
                            # master 브랜치의 deploy.yaml을 gitops 브랜치에 강제 복사
                            echo "=== master 브랜치 deploy.yaml을 gitops 브랜치에 복사 ==="
                            cp ./k8s/deploy.yaml.backup ./k8s/deploy.yaml
                            
                            echo "=== 현재 브랜치 확인 ==="
                            git branch
                            git status
                            
                            # deploy.yaml 이미지 태그 수정
                            echo "=== deploy.yaml 이미지 태그 수정 ==="
                            sed -Ei "s#(image:[[:space:]]*${IMAGE_REGISTRY}/${IMAGE_NAME})[^[:space:]]+#\\1:${FINAL_IMAGE_TAG}#" ./k8s/deploy.yaml
                            
                            echo "=== 최종 수정된 deploy.yaml ==="
                            cat ./k8s/deploy.yaml
                            
                            # 백업 파일 제거
                            rm -f ./k8s/deploy.yaml.backup
                            
                            git add ./k8s/deploy.yaml || true
                            git status
                        """
                        
                        withCredentials([usernamePassword(credentialsId: "${env.GIT_ID}", usernameVariable: 'GIT_PUSH_USER', passwordVariable: 'GIT_PUSH_PASSWORD')]) {
                            sh """
                                set -euxo pipefail  # 디버깅 모드 활성화
                                
                                echo "=== Credentials 확인 ==="
                                echo "GIT_PUSH_USER: \${GIT_PUSH_USER}"
                                echo "Password length: \${#GIT_PUSH_PASSWORD}"
                                echo "gitRepoPath: ${gitRepoPath}"
                                
                                # Git 변경사항 확인
                                if ! git diff --cached --quiet; then
                                    echo "=== Changes detected, committing and pushing to gitops branch ==="
                                    git commit -m "[AUTO] Update deploy.yaml with image ${env.FINAL_IMAGE_TAG}"
                                    
                                    echo "=== Setting remote URL ==="
                                    git remote set-url origin https://\${GIT_PUSH_USER}:\${GIT_PUSH_PASSWORD}@${gitRepoPath}
                                    
                                    echo "=== Pushing to gitops branch ==="
                                    git push origin gitops
                                    echo "=== Push to gitops branch completed successfully ==="
                                else
                                    echo "=== No changes to commit ==="
                                fi
                            """
                        }
                    }
                }
            }
        }


    }
}
