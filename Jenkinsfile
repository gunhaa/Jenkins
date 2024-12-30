// 파이프 라인의 시작
pipeline {
    // 스테이지 별로 다른 거
    // 어떤 노예를 쓸 것인가
    agent any

    //pipeline이 몇분 주기로 가져올 것인가 -> cron syntax
    triggers {
        pollSCM('*/3 * * * *')
    }

    //pipeline안에서 쓸 환경변수를 입혀주는 것
    //EC2 콘솔에서 등록해주어야 한다.
    environment {
      // AWS access key라고 생각하면된다.
      // AWS 명령어를 쓸 수 있도록
      // 시스템 환경변수로 들어간다.
      // AWS IAM(직원들에게 id발급해줄때 쓰는 콘솔)항목에 들어가서 jenkins 사용자를 만든다. (iam-액세스관리-사용자)
      // 기존 정책 직접 연결 - 모든 권한 (실습 중이라서)
      // 만든 계정을 클릭 후 엑세스 키 만들기로 이동
      // AWS 컴퓨팅 서비스에서 실행되는 애플리케이션 선택
      // 설명 태그값 jenkins
      // 액세스 키 ID / 비밀 액세스 키 
      // 젠킨스 콘솔로 이동
      // Add credential 이동
      // secret text 설정 - secret(액세스 키) / ID : awsAccessKeyId 입력
      // Add credential 이동
      // secret text 설정 - secret(비밀 액세스 키) / ID : awsSecretAccessKey 입력
      // 해당 스크립트와 매핑시킨다.

      AWS_ACCESS_KEY_ID = credentials('awsAccessKeyId')
      AWS_SECRET_ACCESS_KEY = credentials('awsSecretAccessKey')
      AWS_DEFAULT_REGION = 'ap-northeast-2'
      HOME = '.' // Avoid npm root owned
    }

    // 
    stages {
        // 레포지토리를 다운로드 받음
        stage('Prepare') {
            agent any
            
            steps {
                echo 'Clonning Repository'

                git url: 'https://github.com/gunhaa/temp.git',
                    branch: 'main',
                    // git과 연결될때 사용한 것의 id를 입력하면 된다.
                    credentialsId: 'gittest'
            }

            post {
                // If Maven was able to run the tests, even if some of the test
                // failed, record the test results and archive the jar file.
                // 보내는 섹션으로, 메일도 보내고 슬랙도 보내는 등 많은 동작을 할 수 있다.
                success {
                    echo 'Successfully pull Repository'
                }

                always {
                  echo "i tried..."
                }

                cleanup {
                  echo "after all other post condition"
                }
            }
        }

        
        stage('Only for production'){
          when {
            // 이 브랜치가 prod이고,
            branch 'production'
                              // app_env가 prod이면, stage를 실행해라 라는 스크립트
            environment name: 'APP_ENV', value: 'prod'
            anyof {
              environment name: 'DEPLOY_TO', value: 'production'
              environment name: 'DEPLOY_TO', value: 'staging'
            }
          }
        }
        
        // aws s3 에 파일을 올림
        stage('Deploy Frontend') {
          steps {
            echo 'Deploying Frontend'
            // 프론트엔드 디렉토리의 정적파일들을 S3 에 올림, 이 전에 반드시 EC2 instance profile 을 등록해야함.
            // 현재 폴더 website안에 있는 파일들을 s3로 옮긴다.
            // s3가 없는 상태이므로, s3를 만들어야 한다.
            // 아마존 콘솔 s3 이동
            // 버킷만들기 - 이름 : 아무거나(전 세계에서 사용해서 겹친다) - 스킵 - 퍼블릭 액세스 차단해제(과금 안되려면 나중에 삭제해야 한다.) 
            // 실제 prod라면 해당 위치에서 프론트엔드 배포를 한다.(lint 등)
            dir ('./website'){
                sh '''
                aws s3 sync ./ s3://gunhatest
                '''
            }
          }

          post {
              // If Maven was able to run the tests, even if some of the test
              // failed, record the test results and archive the jar file.
              success {
                  echo 'Successfully Cloned Repository'
                  // 성공 시 메일 보내기
                  // 메일을 보내기 위해서
                  // Jenkins 콘솔로이동 -> Jenkins 관리 -> System -> Extended E-mail Notification에 입력 or E-mail로 알려줌의 SMTP 서버에서 `smtp.gmail.com` 입력 -> 고급 -> Use SMTP Authentication -> 사용자명/비밀번호 입력(구글)
                  mail  to: 'wh8299@gmail.com',
                        subject: "Deploy Frontend Success",
                        body: "Successfully deployed frontend!"

              }

              failure {
                  echo 'I failed :('

                  mail  to: 'wh8299@gmail.com',
                        subject: "Failed Pipelinee",
                        body: "Something is wrong with deploy frontend"
              }
          }
        }
        
        stage('Lint Backend') {
            // Docker plugin and Docker Pipeline 두개를 깔아야 사용가능!
            agent {
              // 도커를 설치해서 노드 설치
              // 이후 린팅
              docker {
                image 'node:latest'
              }
            }
            
            steps {
              dir ('./server'){
                  // Install necessary Node.js dependencies
                  // Run linting to check for code quality and style issues
                  sh '''
                  npm install&&
                  npm run lint
                  '''
              }
            }
        }
        
        stage('Test Backend') {
          agent {
            docker {
              image 'node:latest'
            }
          }
          steps {
            echo 'Test Backend'

            dir ('./server'){
              // 테스트코드 실행
                sh '''
                npm install
                npm run test
                '''
            }
          }
        }
        
        stage('Bulid Backend') {
          agent any
          steps {
            echo 'Build Backend'

            dir ('./server'){
                sh """
                docker build . -t server --build-arg env=${PROD}
                """
            }
          }

          post {
            // 배포하다 실패하면 여기서 중단시켜야해서, 중단시키는 방법이다.
            failure {
              error 'This pipeline stops here...'
            }
          }
        }
        
        stage('Deploy Backend') {
          agent any

          steps {
            echo 'Build Backend'

            dir ('./server'){
              //docker rm -f $(docker ps -aq) 도커 컨테이너가 있다면 모두 종료시킨다.
                sh '''
                docker run -p 80:80 -d server
                '''
            }
          }

          post {
            success {
              mail  to: 'wh8299@gmail.com',
                    subject: "Deploy Success",
                    body: "Successfully deployed!"
                  
            }
          }
        }
    }
}

// docker plugin을 jenkins에 설치해야한다.
// jenkins plugin manager-> available plugins -> Docker , Docker Pipeline 설치
// jenkins에서 도커를 사용하기위해 AWS EC2 인스턴스에서 `groupadd docker` , `usermod -aG docker jenkins` 를 입력해야한다 (docker에 떠 있는 상태이므로 `docker exec -it jenkins_container_name bash` 입력 후 쉘에서 입력해야한다.)