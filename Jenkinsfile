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
                
                // === 移到这里 ===
                // 趁着容器还在，直接归档。
                // 注意：虽然是在容器里执行，但 Jenkins 会聪明地把容器里的文件传回 Master
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }
}
