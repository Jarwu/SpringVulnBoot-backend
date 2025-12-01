pipeline {
    agent none 

    stages {
        // 第一阶段：编译 (Java 构建)
        stage('Build & Test') {
            agent {
                docker {
                    image 'maven:3.8.6-openjdk-11'
                    args '-v /home/ubuntu/maven_repo:/tmp/m2'
                }
            }
            steps {
                echo '====== 开始编译 ======'
                sh 'mvn clean package -DskipTests -Dmaven.repo.local=/tmp/m2'
                
            }
        }
    
    // 第二阶段：SCA 扫描 (OWASP Dependency Check)
        stage('SCA Analysis') {
            agent {
                docker {
                    // 使用 ODC 官方镜像
                    image 'owasp/dependency-check:latest'
                    // 核心优化：挂载 data 目录做缓存！
                    args '-v /home/ubuntu/odc-data:/usr/share/dependency-check/data/odc-data --entrypoint=""' 
                    // --entrypoint="" 是为了覆盖镜像默认的启动命令，让我们能在 steps 里手动执行
                }
            }
            steps {
                echo '====== 2. 开始组件安全扫描 ======'
                // 执行扫描
                // --scan . : 扫描当前工作区（Jenkins 会自动把代码挂载进来）
                // --format ALL : 生成 HTML, JSON 等所有格式
                // --project "MyProject" : 报告里的项目名
                // --data /usr/share/dependency-check/data/odc-data : 指定数据库位置
                sh '''
                    /usr/share/dependency-check/bin/dependency-check.sh \
                    --scan . \
                    --format HTML \
                    --format JSON \
                    --project "SpringVulnBoot" \
                    --out . \
                    --data /usr/share/dependency-check/data/odc-data
                '''
            }
        }
    }
    // 构建后操作：无论成功失败，都归档产物
    post {
        always {
            echo '====== 3. 归档构建产物与安全报告 ======'
            // 归档 jar 包
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            // 归档 ODC 生成的报告 (HTML 和 JSON)
            archiveArtifacts artifacts: 'dependency-check-report.html, dependency-check-report.json', fingerprint: true
        }
    }
    
}
