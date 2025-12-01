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
                // 【修正点】: 编译完直接在当前容器上下文归档 Jar 包
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    
        // 第二阶段：SCA 扫描 (OWASP Dependency Check)
        stage('SCA: OWASP Dependency Check') {
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
                // allowEmptyArchive: true 是防止万一没生成报告导致流水线报错，虽然一般都会生成
                archiveArtifacts artifacts: 'dependency-check-report.html, dependency-check-report.json', fingerprint: true, allowEmptyArchive: true
            }
        }
        // --- 第三阶段：SCA 扫描 (Trivy - 重点推荐) ---
        stage('SCA: Trivy') {
            agent {
                docker {
                    // 使用 Trivy 官方镜像
                    image 'aquasec/trivy:latest'
                    // 挂载缓存目录，否则每次都会下载漏洞库，很慢！
                    // 【改动1】加上 --entrypoint="" 确保 Jenkins 能完全控制容器
                    // 注意：需要在宿主机创建 /home/ubuntu/trivy_cache 目录并给权限
                    args '-v /home/ubuntu/trivy_cache:/root/.cache/ --entrypoint=""'
                }
            }
            steps {
                echo '====== 3. Trivy 代码依赖扫描 ======'
                
                // 1. 打印表格到控制台（给看日志的人看）
                // --security-checks vuln: 只扫漏洞，不扫配置(config)和密钥(secret)，虽然它也能扫
                // 【改动2】加上 --debug 查看详细日志
                // 【改动3】加上 --timeout 15m 防止网络慢导致超时
                sh '''
                    trivy fs \
                    --debug \
                    --timeout 15m \
                    --security-checks vuln \
                    --format table \
                    .
                '''

                // 2. 生成 JSON 报告（给后续程序处理或存档）
                sh 'trivy fs --security-checks vuln --format json --output trivy-report.json .'
                
                // 3. (进阶) 如果有高危漏洞，直接断掉流水线？
                // sh 'trivy fs --exit-code 1 --severity CRITICAL --security-checks vuln .'

                archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
            }
        }
    }
    
}
