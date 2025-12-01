pipeline {
    agent none  // 不在 Jenkins 主节点运行，强制每个阶段指定 agent

    stages {
        // 阶段 1: 编译构建
        stage('Build & Test') {
            agent {
                docker {
                    // 使用官方 Maven 镜像，带 JDK 11 (根据你项目的 Java 版本调整)
                    image 'maven:3.8.6-openjdk-11' 
                    // 关键：把宿主机的 maven 缓存挂载进去，否则每次都要重新下载半天 jar 包
                    args '-v /home/ubuntu/maven_repo:/tmp/m2'
                }
            }
            steps {
                echo '====== 开始编译 Java 项目 ======'
                // 这里的 mvn 命令是在 maven 容器里执行的
                sh 'mvn clean package -DskipTests -Dmaven.repo.local=/tmp/m2' 
            }
        }
    }
    
    post {
        always {
            echo '====== 构建结束 ======'
        }
        success {
            echo '====== 构建成功！ ======'
        }
        failure {
            echo '====== 构建失败，请检查日志 ======'
        }
    }
}
