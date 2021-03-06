pipeline {
  agent any
  
  options {
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage ('Install docker-compose') {
      steps{
        sh "curl -L https://github.com/docker/compose/releases/download/1.23.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose"
        sh "chmod +x /usr/local/bin/docker-compose"
      }
    }

    stage('Build') {
      steps {
        runCompose("-f docker-compose.yml -f compose/test.yml -f compose/robot.yml", "build --pull")
      }
    }

    stage('Test Components') {
      steps {
        runCompose("-f compose/test.yml", "run jest")
      }
    }

    stage('Lint') {
      steps {
        runCompose("-f compose/test.yml", "run lint")
      }
    }

    stage('Audit') {
      steps {
        runCompose("-f compose/test.yml", "run audit")
      }
    }

    stage('Robot Test') {
      steps {
        runCompose("-f docker-compose.yml -f compose/test.yml -f compose/robot.yml", "down -v")
        runCompose("-f compose/robot.yml", "run robot.backend ./wait-for robot.db:5432 -- npm run db:seed:all")
        runCompose("-f compose/robot.yml", "run robot")
      }
      post {
        always {
          step([$class: 'ArtifactArchiver', artifacts: 'results/robot/log.html, results/robot/selenium-*.png'])
          step([$class: 'RobotPublisher',
              disableArchiveOutput: false,
              logFileName: 'results/robot/log.html',
              onlyCritical: true,
              otherFiles: '',
              outputFileName: 'results/robot/output.xml',
              outputPath: '.',
              passThreshold: 90,
              reportFileName: 'results/robot/report.html',
              unstableThreshold: 100
              ]);
          step([$class: 'InfluxDbPublisher',
              customData: null,
              customDataMap: null,
              customPrefix: null,
              target: 'jenkins_data',
              selectedTarget: 'jenkins_data' // If you have multiple InfluxDB targets configured, selectedTarget should be set as well
            ]);
            /*step([$class: 'XrayImportBuilder',
              endpointName: '/robot', importFilePath: 'results/robot/output.xml',
              importToSameExecution: 'true', projectKey: 'XP',
              serverInstance: 'a8805c85-ed32-453b-b718-302ba21a54e0'])
              */
        }
      }
    }
  }
/*
  post {
    always {
      // notifyBuild(currentBuild.result)
    }
  }
*/
}

def runCompose(composeFiles = "", operation = "build", setEnvironment = "") {
  sh "${setEnvironment} docker-compose -p designsystem --project-directory . ${composeFiles} ${operation}"
}

def runDeploy() {
  sh "bin/deploy.sh"
}

def notifyBuild(String buildStatus = 'STARTED') {
  buildStatus = buildStatus ?: 'SUCCESS'

  def colorCode = '#FF9FA1'
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>"""

  if (buildStatus == 'STARTED') {
    colorCode = '#D4DADF'
  } else if (buildStatus == 'SUCCESS') {
    colorCode = '#BDFFC3'
  } else {
    colorCode = '#FF9FA1'
  }

  slackSend(teamDomain: "", token: "", color: colorCode, message: summary)
}
