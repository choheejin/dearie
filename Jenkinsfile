def envProps
def buildSuccess = false


def generateEnvString = { keys ->
    keys.collect { key -> "${key}=${envProps[key]}" }.join('\n')
}

def generateWithEnvList = { keys ->
    keys.collect { key -> "${key}=${envProps[key]}" }
}

pipeline {
    agent any

    parameters {
  		choice(name: 'ENV', choices: ['develop', 'master'], description: 'Select deploy environment')
	}

    environment {
        MATTERMOST_WEBHOOK_ID = 'MATTERMOST_WEBHOOK'
    }

    stages {
        stage('Decide Environment') {
        // 0. 브랜치 기반 ENV 자동 설정
            steps {
                script {
                    def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                    def selectedEnv = params.ENV?.trim()?.toLowerCase()

                    if (!selectedEnv || !(selectedEnv in ['develop', 'master'])) {
                        selectedEnv = (branch == 'develop') ? 'develop' : 'master'
                        echo "🔄 ENV auto-detected as: ${selectedEnv}"
                    } else {
                        echo "✅ ENV manually selected: ${selectedEnv}"
                    }
                    env.ENV = selectedEnv
                }
            }
        }

        // 1. 먼저 .env 파일부터 읽음
        stage('Load .env File') {
            steps {
                withCredentials([string(credentialsId: 'soboro-dotenv', variable: 'DOTENV')]) {
                    script {
                        def envFilePath = "${env.WORKSPACE}/cicd/.env"
                        
                        def correctedContent = DOTENV.replaceAll(/([A-Z][A-Z0-9_]+)=/, '\n$1=').trim()
                
                        writeFile file: envFilePath, text: correctedContent
                
                        // 이제 제대로 파싱
                        envProps = [:]
                        correctedContent.readLines().each { line ->
                            if (line && line.contains('=') && !line.trim().startsWith('#')) {
                                def split = line.split('=', 2)
                                    if (split.length == 2) {
                                        envProps[split[0].trim()] = split[1].trim()
                                    }
                                }
                            }
                            
                        echo "✅ .env 파일 읽기 완료: ${envProps.size()}개 프로퍼티"
                        // echo "✅ 키 목록: ${envProps.keySet()}"
                    }
                }
            }
        }
        
        // 2. generate env - backend
        stage('Generate .env') {
            steps {
                script {
                    def requiredVars = [
                        'DEARIE_DB_URL', 'DEARIE_DB_USER', 'DEARIE_DB_PASSWORD', 'DEARIE_DB_NAME', 'DEARIE_JWT_SECRET',
                        'LIGHT_DB_URL', 'LIGHT_DB_USER', 'LIGHT_DB_PASSWORD', 'LIGHT_DB_NAME', 'LIGHT_JWT_SECRET',
                        'KAFKA_BOOTSTRAP_SERVERS', 'KAFKA_TOPIC_NAME', 'KAFKA_CONSUMER_GROUP_ID',
                        'OPENAI_API_KEY', 'S3_ACCESS_KEY', 'S3_SECRET_KEY', 'S3_BUCKET_DEARIE', 'S3_BUCKET_LIGHTREBORN',
                        'NEXT_PUBLIC_NAVER_CLIENT_ID', 'KAKAO_REST_API_KEY'
                    ]

                    requiredVars.each { var ->
                        if (!envProps.containsKey(var)) {
                            error "❌ 필수 변수 ${var}가 envProps에 없습니다."
                        }
                    }

                    def newEnvContent = generateEnvString(requiredVars) + '\nspring.profiles.active=prod'

                    writeFile file: "${env.WORKSPACE}/cicd/.env", text: newEnvContent.trim()
                    echo "✅ .env 재생성 완료"
                }
            }
        }

        // generate env - lightreborn-frontend
        stage('Generate frontend .env.production') {
            steps {
                script {
                    def frontendEnv = """
                    NEXT_PUBLIC_NAVER_CLIENT_ID=${envProps.NEXT_PUBLIC_NAVER_CLIENT_ID}
                    NEXT_PUBLIC_API_URL=/api/dashboard/
                    """.stripIndent().trim()

                    writeFile file: "${env.WORKSPACE}/lightreborn/frontend/.env.production", text: frontendEnv
                    echo "✅ lightreborn frontend용 .env.production 생성 완료"
                }
            }
        }

        // generate env - dearie-frontend
        stage('Generate frontend .env.dearie.production') {
            steps {
                script {
                    def frontendEnv = """
                    NEXT_PUBLIC_BASE_PATH=/dearie
                    NEXT_PUBLIC_API_URL=/api/app/
                    NEXT_PUBLIC_MAPBOX_TOKEN=${envProps.NEXT_PUBLIC_MAPBOX_TOKEN}
                    NEXT_PUBLIC_MAPTILER_KEY=${envProps.NEXT_PUBLIC_MAPTILER_KEY}
                    NEXT_PUBLIC_NAVER_CLIENT_ID=${envProps.NEXT_PUBLIC_NAVER_CLIENT_ID}
                    KAKAO_REST_API_KEY=${envProps.KAKAO_REST_API_KEY}
                    """.stripIndent().trim()

                    writeFile file: "${env.WORKSPACE}/dearie/frontend/.env.dearie.production", text: frontendEnv
                    echo "✅ dearie frontend용 .env.dearie.production 생성 완료"
                }
            }
        }


        // 3. 기존 컨테이너 정리
        stage('Clean up Existing Containers') {
            steps {
                script {
                    def composePath = "${env.WORKSPACE}/docker-compose.yml"
                    def envPath = "${env.WORKSPACE}/cicd/.env"

                    sh """
                        echo \"🧹 docker-compose down (remove orphans)\"
                        docker-compose --env-file ${envPath} -f ${composePath} down --remove-orphans || true
                      
                        docker rm -f dearie-backend lightreborn-backend dearie-frontend lightreborn-frontend || true
                    """
                }
            }
        }

        // 4. 빌드 및 배포
        stage('Docker Compose Up') {
            steps {
                script {
                    def composePath = "${env.WORKSPACE}/docker-compose.yml"
                    def envPath = "${env.WORKSPACE}/cicd/.env"

                    def runtimeEnvKeys = [
                        'DEARIE_DB_URL', 'DEARIE_DB_USER', 'DEARIE_DB_PASSWORD', 'DEARIE_DB_NAME', 'DEARIE_JWT_SECRET',
                        'LIGHT_DB_URL', 'LIGHT_DB_USER', 'LIGHT_DB_PASSWORD', 'LIGHT_DB_NAME', 'LIGHT_JWT_SECRET',
                        'KAFKA_BOOTSTRAP_SERVERS', 'KAFKA_TOPIC_NAME', 'KAFKA_CONSUMER_GROUP_ID',
                        'OPENAI_API_KEY',
                        'S3_ACCESS_KEY', 'S3_SECRET_KEY', 'S3_BUCKET_DEARIE','S3_BUCKET_LIGHTREBORN',
                        'NEXT_PUBLIC_NAVER_CLIENT_ID',
                        'NEXT_PUBLIC_MAPTILER_KEY', 'NEXT_PUBLIC_MAPBOX_TOKEN'
                    ]

                    withEnv(generateWithEnvList(runtimeEnvKeys)) {
                        sh """
                            docker-compose --env-file ${envPath} -f ${composePath} up -d --build
                        """
                    }
                }
            }
        }
        
        // 5. Flyway 데이터 마이그레이션
        stage('Flyway Check and Migration') {
            steps {
                script {
                    echo "🔍 Jenkins Workspace: ${env.WORKSPACE}"
                    
                    def projects = ['dearie', 'lightreborn']
                    
                    projects.each { project ->
                        def projUpper = project.toUpperCase()
                        
                        def migrationPath = (params.ENV == 'master') ?
                            "${env.WORKSPACE}/${project}/backend/src/main/resources/db/migration_master" :
                            "${env.WORKSPACE}/${project}/backend/src/main/resources/db/migration"
                        
                        echo "🔍 Full Migration Path: ${migrationPath}"
                        
                        def networkName = "${project}-net"
                        def dbHost = "${project}-db"
                        def dbUser = envProps.get("${projUpper}_DB_USER") ?: "ssafy"
                        def dbPassword = envProps.get("${projUpper}_DB_PASSWORD") ?: "ssafy"
                        def dbName = project
                        def buildNumber = env.BUILD_NUMBER
                        def tempDir = "/tmp/flyway_sql_${project}_${buildNumber}"
                        
                        // 쉘 스크립트에서 $ 기호 이스케이프 처리
                        sh """#!/bin/bash
                            echo "🔍 환경 변수 확인:"
                            echo "- 프로젝트: ${project}"
                            echo "- 워크스페이스: ${env.WORKSPACE}"
                            echo "- 마이그레이션 경로: ${migrationPath}"
                            echo "- 네트워크: ${networkName}"
                            echo "- DB 호스트: ${dbHost}"
                            echo "- 빌드 번호: ${buildNumber}"
                            echo "- 임시 디렉토리: ${tempDir}"
                            
                            # 마이그레이션 경로가 존재하는지 확인
                            if [ ! -d "${migrationPath}" ]; then
                                echo "⚠️ 마이그레이션 경로가 존재하지 않습니다: ${migrationPath}"
                                exit 0
                            fi
                            
                            # 경로 내용 확인
                            echo "📋 마이그레이션 경로 내용:"
                            ls -la "${migrationPath}"
                            
                            # SQL 파일 검색 및 카운트
                            SQL_FILES=\$(find "${migrationPath}" -name "*.sql" 2>/dev/null | sort)
                            FILE_COUNT=\$(echo "\$SQL_FILES" | grep -v "^\$" | wc -l)
                            
                            if [ \$FILE_COUNT -eq 0 ]; then
                                echo "⚠️ SQL 파일을 찾을 수 없습니다: ${migrationPath}"
                                exit 0
                            fi
                            
                            echo "🚀 파일 \$FILE_COUNT개가 발견되었습니다."
                            echo "📋 SQL 파일 목록:"
                            echo "\$SQL_FILES"
                            
                            # 임시 디렉토리 생성 및 파일 복사 전 정리
                            rm -rf "${tempDir}"
                            mkdir -p "${tempDir}"
                            
                            # Flyway 스키마 히스토리 확인
                            echo "Flyway 스키마 히스토리 확인:"
                            HAS_HISTORY=\$(docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "SELECT EXISTS (SELECT FROM information_schema.tables WHERE table_name = 'flyway_schema_history');" -t | tr -d '[:space:]')
                            
                            # 스키마 히스토리 테이블이 있는지 확인하고 최대 버전 가져오기
                            MAX_VERSION=0
                            if [ "\$HAS_HISTORY" = "t" ]; then
                                MAX_VERSION=\$(docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "SELECT COALESCE(MAX(CAST(version AS INTEGER)), 0) FROM flyway_schema_history WHERE success = true;" -t | tr -d '[:space:]')
                                echo "📊 현재 DB 최대 버전: \$MAX_VERSION"
                            else
                                echo "📊 Flyway 히스토리 테이블이 없습니다. 첫 마이그레이션으로 처리합니다."
                            fi
                            
                            # 파일 복사 및 이름 조정 (버전 순서 맞추기)
                            NEXT_VERSION=\$((\$MAX_VERSION + 1))
                            echo "📊 다음 마이그레이션 시작 버전: \$NEXT_VERSION"
                            
                            # 각 파일을 개별적으로 복사 (절대 경로 사용)
                            COUNTER=\$NEXT_VERSION
                            echo "\$SQL_FILES" | while read file; do
                                if [ -f "\$file" ]; then
                                    # 기존 파일명 추출
                                    filename=\$(basename "\$file")
                                    
                                    # 버전 패턴 확인 (V숫자__)
                                    if [[ "\$filename" =~ ^V[0-9]+__ ]]; then
                                        # 버전 번호 추출 (sed 패턴 수정)
                                        original_version=\$(echo "\$filename" | sed -E 's/V([0-9]+)__.*$/\\1/')
                                        
                                        # 파일 내용 패턴 유지하며 버전만 변경
                                        new_filename=\$(echo "\$filename" | sed -E "s/V[0-9]+__/V\${COUNTER}__/")
                                        echo "🔄 버전 조정: \$filename -> \$new_filename (원본 버전: \$original_version, 새 버전: \$COUNTER)"
                                        cp "\$file" "${tempDir}/\$new_filename"
                                        COUNTER=\$((\$COUNTER + 1))
                                    else
                                        # 버전 형식이 아닌 경우 그대로 복사
                                        echo "📄 일반 파일 복사: \$filename"
                                        cp "\$file" "${tempDir}/\$filename"
                                    fi
                                    
                                    echo "📄 복사됨: \$file -> ${tempDir}/\$(basename "\$file")"
                                fi
                            done
                            
                            # 복사된 파일 목록 확인
                            echo "📋 임시 디렉토리 내용:"
                            ls -la "${tempDir}"
                            
                            # 파일 내용 확인 (5줄만)
                            echo "📄 SQL 파일 내용 샘플:"
                            for f in \$(find "${tempDir}" -name "*.sql" | sort); do
                                echo "===== \$f ====="
                                head -n 5 "\$f"
                                echo "..."
                            done
                            
                            # 볼륨 마운트 테스트
                            echo "🔍 볼륨 마운트 테스트:"
                            docker run --rm -v "${tempDir}:/flyway/sql" alpine ls -la /flyway/sql
                            
                            # 현재 DB 상태 확인
                            echo "🔍 현재 DB 상태 확인:"
                            echo "테이블 목록:"
                            docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "\\\\dt" 2>/dev/null || echo "테이블 목록 조회 실패"
                            
                            # Flyway 스키마 히스토리 확인
                            echo "Flyway 스키마 히스토리:"
                            docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "SELECT * FROM flyway_schema_history ORDER BY installed_rank;" 2>/dev/null || echo "flyway_schema_history 테이블이 없습니다."
                            
                            # Flyway 정보 확인
                            echo "🔍 Flyway 정보:"
                            docker run --rm \\
                                --network "${networkName}" \\
                                -v "${tempDir}:/flyway/sql" \\
                                flyway/flyway \\
                                -locations=filesystem:/flyway/sql \\
                                -url=jdbc:postgresql://${dbHost}:5432/${dbName} \\
                                -user=${dbUser} \\
                                -password=${dbPassword} \\
                                info
                            
                            # Flyway 마이그레이션 실행
                            echo "📦 Flyway 마이그레이션 실행 중..."
                            MIGRATE_RESULT=\$(docker run --rm \\
                                --network "${networkName}" \\
                                -v "${tempDir}:/flyway/sql" \\
                                flyway/flyway \\
                                -locations=filesystem:/flyway/sql \\
                                -url=jdbc:postgresql://${dbHost}:5432/${dbName} \\
                                -user=${dbUser} \\
                                -password=${dbPassword} \\
                                -baselineOnMigrate=true \\
                                migrate 2>&1)
                            
                            MIGRATE_STATUS=\$?
                            echo "\$MIGRATE_RESULT"
                            
                            # 마이그레이션 실패 시 처리
                            if [ \$MIGRATE_STATUS -ne 0 ]; then
                                echo "⚠️ 마이그레이션 실패! (종료 코드: \$MIGRATE_STATUS)"
                                
                                # 마이그레이션 후 DB 상태 확인
                                echo "🔍 마이그레이션 후 DB 상태 확인:"
                                docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "\\\\dt" 2>/dev/null || echo "테이블 목록 조회 실패"
                                
                                echo "🔄 재시도: 마이그레이션 문제 해결을 위한 추가 조치"
                                
                                # 마이그레이션 문제 판단
                                if [[ "\$MIGRATE_RESULT" == *"No migrations found"* ]] || [[ "\$MIGRATE_RESULT" == *"no migration could be resolved"* ]]; then
                                    echo "🔄 리셋 후 재시도: 버전 충돌 문제로 판단됩니다."
                                    
                                    # 안전한 방법으로 강제 리셋 시도
                                    echo "🔄 Flyway 리페어 시도..."
                                    docker run --rm \\
                                        --network "${networkName}" \\
                                        -v "${tempDir}:/flyway/sql" \\
                                        flyway/flyway \\
                                        -locations=filesystem:/flyway/sql \\
                                        -url=jdbc:postgresql://${dbHost}:5432/${dbName} \\
                                        -user=${dbUser} \\
                                        -password=${dbPassword} \\
                                        repair -cleanDisabled=false
                                    
                                    # 모든 SQL 파일 강제 실행
                                    echo "🔄 SQL 직접 실행 시도..."
                                    for f in \$(find "${tempDir}" -name "*.sql" | sort); do
                                        echo "실행: \$f"
                                        cat "\$f" | docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} || echo "SQL 실행 실패: \$f"
                                    done
                                    
                                    # 실행 후 마이그레이션 상태 기록 (수동으로)
                                    echo "🔄 마이그레이션 히스토리 수동 업데이트 시도..."
                                    for f in \$(find "${tempDir}" -name "*.sql" | sort); do
                                        filename=\$(basename "\$f")
                                        
                                        # 버전 추출 (단순화된 방식)
                                        version=\$(echo "\$filename" | grep -oE 'V[0-9]+' | sed 's/V//')
                                        
                                        # 설명 추출 (단순화된 방식)
                                        description=\$(echo "\$filename" | sed 's/V[0-9]*__//g' | sed 's/\.sql$//g')
                                        
                                        # 중복 체크 후 히스토리 추가
                                        docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "
                                            INSERT INTO flyway_schema_history (installed_rank, version, description, type, script, checksum, installed_by, installed_on, execution_time, success)
                                            SELECT 
                                                COALESCE(MAX(installed_rank), 0) + 1,
                                                '\$version',
                                                '\$description',
                                                'SQL',
                                                '\$filename',
                                                0,
                                                '${dbUser}',
                                                NOW(),
                                                0,
                                                true
                                            FROM flyway_schema_history
                                            WHERE NOT EXISTS (
                                                SELECT 1 FROM flyway_schema_history WHERE version = '\$version'
                                            )
                                        " || echo "히스토리 업데이트 실패: \$filename"
                                    done
                                fi
                            else
                                echo "✅ 마이그레이션 성공!"
                                
                                # 마이그레이션 후 DB 상태 확인
                                echo "🔍 마이그레이션 후 최종 상태:"
                                docker exec -i ${dbHost} psql -U ${dbUser} -d ${dbName} -c "SELECT * FROM flyway_schema_history ORDER BY installed_rank;" 2>/dev/null || echo "히스토리 조회 실패"
                            fi
                            
                            # 개발 환경일 경우 임시 디렉토리 보존
                            if [ "${params.ENV}" = "develop" ]; then
                                echo "🛠 개발 환경 - 임시 디렉토리 보존: ${tempDir}"
                            else
                                echo "🧹 임시 디렉토리 정리: ${tempDir}"
                                rm -rf "${tempDir}"
                            fi
                        """
                    }
                }
            }
        }

        // 7. 빌드 성공 여부 상태 반영
        stage('Mark Image Build Success') {
            steps {
                script {
                    buildSuccess = true
                    echo "🫠 현재 빌드 상태: ${currentBuild.result}"
                    echo "✅ 이미지 빌드 성공 상태로 설정: ${buildSuccess}"
                }
            }
        }
    }

    post {
        always {
            script {
                def sendMessage = { String msg ->
                    def payload = groovy.json.JsonOutput.toJson([text: msg])
                    writeFile file: 'payload.json', text: payload

                    withCredentials([string(credentialsId: MATTERMOST_WEBHOOK_ID, variable: 'MATTERMOST_WEBHOOK')]) {
                        sh '''
                            curl -X POST -H 'Content-Type: application/json' -d @payload.json $MATTERMOST_WEBHOOK
                        '''
                    }
                }

                if (buildSuccess || currentBuild.result == 'SUCCESS') {
                    sendMessage("🎉 배포 성공 : `${env.ENV}` 환경\n- Job: `${env.JOB_NAME}`\n- Build: #${env.BUILD_NUMBER}")
                } else {
                    sendMessage("❌ 배포 실패 : `${env.ENV}` 환경\n- Job: `${env.JOB_NAME}`\n- Build: #${env.BUILD_NUMBER}\n- [로그 확인하기](${env.BUILD_URL})")
                }

                // 컨테이너가 안정화된 후에 .env 파일 삭제
                sh """
                    echo "🧹 보안상 민감한 파일 정리 중..."
                    find . -name ".env" -type f -delete 2>/dev/null || true
                    find . -name ".env.production" -type f -delete 2>/dev/null || true
                    find . -name ".env.dearie.production" -type f -delete 2>/dev/null || true
                    rm -f payload.json 2>/dev/null || true
                """
            }
        }

        success {
            script {
                if (params.ENV == 'master') {
                    echo '🎉 Build 성공 → Stable 이미지 태깅 및 푸시'
                    sh '''
                        # backend
                        docker tag dearie-backend dearie-backend:stable
                        docker tag lightreborn-backend lightreborn-backend:stable

                        # frontend
                        docker tag dearie-frontend dearie-frontend:stable
                        docker tag lightreborn-frontend lightreborn-frontend:stable

                        # build all
                        docker build -t dearie-backend:stable .
                        docker build -t lightreborn-backend:stable .
                        docker build -t dearie-frontend:stable .
                        docker build -t lightreborn-frontend:stable .
                    '''
                }
            }
        }

        failure {
            script {
                if (params.ENV == 'master') {
                    echo '⛔ 실패 → 이전 stable 이미지로 롤백 시도'
                    sh '''
                        # stop & remove
                        docker stop dearie-backend || true
                        docker stop lightreborn-backend || true
                        docker stop dearie-frontend || true
                        docker stop lightreborn-frontend || true

                        docker rm dearie-backend || true
                        docker rm lightreborn-backend || true
                        docker rm dearie-frontend || true
                        docker rm lightreborn-frontend || true

                        # run rollback
                        docker run -d --name dearie-backend --network dearie-net -p 8082:8082 dearie-backend:stable
                        docker run -d --name lightreborn-backend --network lightreborn-net -p 8081:8081 lightreborn-backend:stable
                        docker run -d --name dearie-frontend --network dearie-net  -p 3001:3001 dearie-frontend:stable
                        docker run -d --name lightreborn-frontend --network lightreborn-net -p 3000:3000 lightreborn-frontend:stable
                    '''
                }
            }
        }
    }
}
