def call(body) {
  def pipelineParams = [:]
  body.resolveStrategy = Closure.DELEGATE_FIRST
  body.delegate = pipelineParams
  body()

  pipeline {
    agent {
      label 'builder-backend'
    }
    options {
      disableConcurrentBuilds()
      skipStagesAfterUnstable()
      skipDefaultCheckout(true)
      buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
      stage('build') {
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          deleteDir()
          checkout scm
          sh """
            make ci-build
          """
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
      stage('qa: static code analysis') {
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          sh """
            make ci-static-code-analysis
          """
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
      stage('qa: unit & integration tests') {
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          sh """
            make ci-tests
          """
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
      stage('approval: dev') {
        agent none
        when {
          not {
            branch 'master'
          }
        }
        steps {
          script {
            timeout(5) {
              input "Deploy to dev?"
            }
          }
        }
      }
      stage('deploy: dev') {
        when {
          allOf {
            not {
              branch 'master'
            }
            expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
          }
        }
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          releaseHeroku(pipelineParams.developmentRepositoryUrl, 'development')
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
      stage('deploy: stg') {
        when {
          allOf {
            branch 'master'
            expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
          }
        }
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          releaseHeroku(pipelineParams.stagingRepositoryUrl, 'staging')
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
      stage('approval: prod') {
        agent none
        when {
          branch 'master'
        }
        steps {
          script {
            timeout(5) {
              input "Deploy to production?"
            }
          }
        }
      }
      stage('deploy: prod') {
        when {
          allOf {
            branch 'master'
            expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
          }
        }
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          releaseHeroku(pipelineParams.productionRepositoryUrl, 'production')
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
      stage('publish api docs: prod') {
        when {
          allOf {
            branch 'master'
            expression { return currentBuild.result == null || currentBuild.result == 'SUCCESS' }
          }
        }
        steps {
          notifyBitbucket('INPROGRESS', env.STAGE_NAME, env.STAGE_NAME)
          
          sh """
            make docs ${pipelineParams.swaggerYamlFilename}
          """
        }
        post {
          success {
            notifySuccess(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
          failure {
            notifyFailure(env.STAGE_NAME, pipelineParams.slackNotificationChannel)
          }
        }
      }
    }
    post {
      always {
        script {
          if (currentBuild.result == null) {
            currentBuild.result = 'SUCCESS'
          }
        }
        step([$class: 'Mailer', notifyEveryUnstableBuild: true, recipients: gitAuthorEmail(), sendToIndividuals: true])
      }
    }
  }
}
