pipeline {
    agent none 

    stages {
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
    }

    // === 新增部分开始 ===
    post {
        success {
            // 只有构建成功时，才归档 target 目录下的 jar 包
            // fingerprint: true 表示给文件算个哈希值，方便追踪
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            echo '====== 产物已归档，请在 Jenkins 页面查看 ======'
        }
    }
    // === 新增部分结束 ===
}
