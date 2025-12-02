pipeline {
    agent none 

    stages {
        // 阶段一：编译 + SAST 扫描 (合并在一起效率最高)
        stage('Build & SAST (SonarQube)') {
            agent {
                docker {
                    // 依然使用 Maven 镜像
                    image 'maven:3.8.6-openjdk-11'
                    // 挂载 m2 缓存，加速构建
                    // 【关键】需要把当前容器加入到 devops-net 网络，否则连不上 SonarQube
                    args '-v /home/ubuntu/maven_repo:/tmp/m2 --network devops-net'
                }
            }
            steps {
                echo '====== 1. 编译并进行 SonarQube 代码分析 ======'
                
                // withSonarQubeEnv 里的名字必须和 Jenkins 系统配置里的 Name 一致
                withSonarQubeEnv('sonar-server') {
                    // mvn sonar:sonar 会自动读取环境变量里的配置连接服务端
                    // -Dsonar.projectKey 必须和 SonarQube 里创建的一致
                    sh '''
                        mvn clean package sonar:sonar \
                        -Dmaven.repo.local=/tmp/m2 \
                        -DskipTests \
                        -Dsonar.projectKey=SpringVulnBoot
                    '''
                }
                
                // 归档构建产物
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        // 阶段二：质量门禁 (Quality Gate) - 这才是 DevSecOps 的核心
        // 注意：这个步骤不需要在 Docker 代理里跑，Jenkins 主节点跑就行
        stage("Quality Gate") {
            agent any
            steps {
                echo '====== 2. 检查 SonarQube 质量门禁结果 ======'
                timeout(time: 5, unit: 'MINUTES') {
                    // waitForQualityGate 会挂起流水线，等待 SonarQube 分析完成并返回结果
                    // 如果结果是 ERROR (比如漏洞太多)，流水线会在这里失败
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        // 编译 (Java 构建)
        //stage('Build & Test') {
        //    agent {
        //         docker {
        //             image 'maven:3.8.6-openjdk-11'
        //             args '-v /home/ubuntu/maven_repo:/tmp/m2'
        //         }
        //     }
        //     steps {
        //         echo '====== 开始编译 ======'
        //         sh 'mvn clean package -DskipTests -Dmaven.repo.local=/tmp/m2'
        //         // 【修正点】: 编译完直接在当前容器上下文归档 Jar 包
        //         archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        //     }
        // }
    
        // SCA 扫描 (OWASP Dependency Check)
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
        //SCA 扫描 (Trivy - 重点推荐) ---
        stage('SCA: Trivy') {
            agent {
                docker {
                    // 使用 Trivy 官方镜像
                    image 'aquasec/trivy:latest'
                    // 挂载缓存目录，否则每次都会下载漏洞库，很慢！
                    // 【改动1】加上 --entrypoint="" 确保 Jenkins 能完全控制容器
                    // 注意：需要在宿主机创建 /home/ubuntu/trivy_cache 目录并给权限
                    //【关键修改 2】 容器内的挂载点改为了 /tmp/trivy_cache (大家都可写)
                    args '-v /home/ubuntu/trivy_cache:/tmp/trivy_cache --entrypoint=""'
                }
            }
            // 【关键修改 3】 设置环境变量，告诉 Trivy 改变缓存位置
            environment {
                TRIVY_CACHE_DIR = "/tmp/trivy_cache"
            }
            steps {
                echo '====== 3. Trivy 代码依赖扫描 ======'
                
                // 1. 打印表格到控制台（给看日志的人看）
                // --security-checks vuln: 只扫漏洞，不扫配置(config)和密钥(secret)，虽然它也能扫
                // 【改动2】加上 --debug 查看详细日志
                // 【改动3】加上 --timeout 15m 防止网络慢导致超时
                // 检查一下现在的身份（验证用，可选）
                sh 'id' 
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
