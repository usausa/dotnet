# CI Jenkins

## Web

```gradle
pipeline {
  agent any

  parameters {
    booleanParam(name: 'CLEAN', defaultValue: false)
  }

  options {
    ansiColor('xterm')
    timestamps()

    timeout(time: 5, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '50', artifactNumToKeepStr: '50'))
  }

  triggers {
    pollSCM('''0-59/10 0-1 * * *
0-59/10 6-23 * * *''')
  }

  stages {
    stage('Checkout') {
      steps {
        script {
          if (env.CLEAN) {
            cleanWs()
          }
        }

        checkout([
          $class: 'GitSCM',
          branches: [[name: '**']],
          userRemoteConfigs: [[url: 'https://server/project/template.git']]])
      }
    }

    stage('Build') {
      steps {
        bat '''
rmdir /s /q R:\\Temp\\Template
mkdir R:\\Temp\\Template
SET TEMP=R:\\Temp\\Template
SET TMP=R:\\Temp\\Template

dotnet clean Template.sln -c Release -nodeReuse:false
dotnet build Template.sln -c Release -nodeReuse:false

rmdir /s /q Publish
dotnet publish --no-restore Template.Web\\Template.Web.csproj -o Publish\\Web -c Release -nodeReuse:false
'''
      }
    }

    stage('Inspect') {
      steps {
        bat '''
SET TEMP=R:\\Temp\\Template
SET TMP=R:\\Temp\\Template

jb inspectcode Template.sln -o=results.xml --no-build --no-swea --properties:Configuration=Release
'''
      }
    }

    stage('Test') {
      steps {
        bat '''
SET TEMP=R:\\Temp\\Template
SET TMP=R:\\Temp\\Template

rmdir /s /q TestResults
dotnet test --no-build Template.sln -c Release -nodeReuse:false -l:"trx;LogFileName=.\\TestResults\\UnitTests.trx" /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura /p:Exclude=\\"[Template.Core.DataAccessor]*,[Template.Web.Views]*\\"
'''
      }
    }

    stage('Report') {
      steps {
        // Cobertura coverage report
        step([
          $class: 'CoberturaPublisher',
          coberturaReportFile: '**/*.cobertura.xml',
          failNoReports :true,
          sourceEncoding: 'UTF_8'])

        // Publish MSTest test result report
        step([
          $class: 'MSTestPublisher',
          testResultsFile:"**/*.trx",
          failOnError: true,
          keepLongStdio: false])

        // Record compiler warnings and static analysis results
        recordIssues(
          tools: [
            msBuild(reportEncoding: 'UTF-8'),
            resharperInspectCode(pattern: 'results.xml', reportEncoding: 'UTF-8')],
          sourceCodeEncoding: 'UTF-8',
          trendChartType: 'TOOLS_ONLY',
          qualityGates: [[threshold: 1, type: 'TOTAL', unstable: true]])
        recordIssues(
          tools: [taskScanner(includePattern: '**/*.cs', lowTags: 'TODO')],
          sourceCodeEncoding: 'UTF-8',
          trendChartType: 'TOOLS_ONLY')

        // Step counter
        stepcounter(
          outputFile: 'stepcounter.csv',
          outputFormat: 'csv',
          settings: [[
              key: 'Core',
              filePattern: 'Template.Core/**/*.cs',
              filePatternExclude: 'Template.Core/obj/**/*.cs',
              encoding: 'UTF-8'
            ],[
              key: 'Web',
              filePattern: 'Template.Web/**/*.cs',
              filePatternExclude: 'Template.Web/obj/**/*.cs',
              encoding: 'UTF-8'
            ],[
              key: 'Test',
              filePattern: 'Template.Tests/**/*.cs',
              filePatternExclude: 'Template.Tests/obj/**/*.cs',
              encoding: 'UTF-8']])
        bat(script: 'StepCounterSummary stepcounter.csv stepcounter-summary.csv')

        // Plot
        plot(
          csvFileName: 'stepcounter-summary.csv',
          csvSeries: [[file: 'stepcounter-summary.csv']],
          group: 'StepCounter',
          title: 'Step',
          style: 'line')
      }
    }
  }

  post {
    success {
      // Send build artifacts to a windows share
      script {
        if (isReleaseBranch(getCurrentBranch())) {
          cifsPublisher(
            publishers: [[
              configName: 'Artifact', transfers: [[
                  sourceFiles: 'Publish/**/*',
                  removePrefix: 'Publish/',
                  remoteDirectory: 'Template',
                  excludes: '**/*.pdb,**/appsettings.Development.json',
                  patternSeparator: '[, ]+',
                  cleanRemote: true]]]])
        }
      }

      // Slack notifications
      postSlackSend('good', 'Success')
    }

    unstable {
      // Slack notifications
      postSlackSend('warning', 'Unstable')
    }

    failure {
      // Slack notifications
      postSlackSend('danger', 'Failure')
    }
  }
}

def getCurrentBranch() {
  def lines = bat(
    script: 'git name-rev --name-only HEAD',
    returnStdout: true
  ).split('\n')
  lines[lines.size() - 1]
}

def isReleaseBranch(branch) {
  branch.contains('master') || branch.contains('main') || ('tags')
}

def postSlackSend(color, result) { 
  slackSend(color: color, message: "${env.JOB_NAME} #${env.BUILD_NUMBER} ${result} [${currentBuild.durationString}] (<${env.BUILD_URL}|Open>)")
}
```
