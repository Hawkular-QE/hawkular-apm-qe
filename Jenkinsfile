def jenkinsNode = env.JAEGER_SLAVE ?: "master"
node (jenkinsNode) {
    // Default ports for the Jaeger all-in-one docker image
    def dockerUdpPort = "5775"
    def dockerApiPort = "16686"

    // Set these values from environment variables if they exist
    def jaegerServerHost = env.JAEGER_SERVER_HOST ?: "localhost"
    def jaegerUdpPort = env.JAEGER_UDP_PORT ?: dockerUdpPort
    def jaegerApiPort = env.JAEGER_API_PORT ?: dockerApiPort

    echo "Using udp port " + jaegerUdpPort + " api port " + jaegerApiPort

    stage('Setup environment') {
        // Set description here if desired
        currentBuild.description = "something...."
        git branch: 'jaeger', url: 'https://github.com/Hawkular-QE/hawkular-apm-qe.git'

        // Add tools to the environment and path
        def M2_HOME = tool 'Maven-3.3.9'
        def JAVA_HOME = tool 'Oracle-jdk8'
        env.M2_HOME = "${M2_HOME}"
        env.JAVA_HOME = "${JAVA_HOME}"
        env.PATH = "${M2_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
    }
    stage('Kill Existing Instance') {
        sh "docker ps | grep jaegertracing  | awk '{print \$1}' | xargs docker stop || true"
    }
    stage('Startup Jaeger Server') {
        sh "BUILD_ID=dontKillMe nohup docker run -d -p" + jaegerUdpPort + ":" + dockerUdpPort +"/udp -p" + jaegerApiPort +":" + dockerApiPort + " jaegertracing/all-in-one:latest &"
    }
    stage('Wait until server is available') {
        timeout(time: 2, unit: 'MINUTES') {
            waitUntil {
                def r = sh script: 'wget -q http://' + jaegerServerHost + ':' + jaegerApiPort + '/api/services -O /dev/null', returnStatus: true
                return (r == 0);
            }
        }

        sleep 5 // FIXME Do we really need this?
    }
    stage('Build and run tests') {
        sh "mvn -Dmaven.test.failure.ignore clean test"
    }
    stage('Record test results') {
        junit '**/target/surefire-reports/TEST-*.xml'
    }
    stage('Docker shutdown') {
        sh "docker ps | grep jaegertracing  | awk '{print \$1}' | xargs docker stop || true"
        sh "docker ps"
    }

}