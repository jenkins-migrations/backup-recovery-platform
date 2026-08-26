<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: MSBuild for .NET projects
node {
    stage('Checkout') {
        checkout scm
    }

    stage('Restore') {
        bat 'nuget restore SolutionName.sln'
    }

    stage('Build') {
        bat "\"${tool 'MSBuild'}\" SolutionName.sln /p:Configuration=Release /p:Platform=\"Any CPU\" /p:ProductVersion=1.0.0.${env.BUILD_NUMBER}"
    }

    stage('Test') {
        bat "\"${tool 'VSTest'}\" ProjectName.Tests\\bin\\Release\\ProjectName.Tests.dll"
        publishTestResults testResultsPattern: '**/*.trx'
    }

    stage('Archive') {
        archiveArtifacts artifacts: 'ProjectName/bin/Release/**/*', fingerprint: true
        publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: true,
            keepAll: true,
            reportDir: 'ProjectName/bin/Release',
            reportFiles: 'index.html',
            reportName: 'Build Report'
        ])
    }

    stage('Deploy') {
        when {
            branch 'master'
        }
        steps {
            bat 'xcopy ProjectName\\bin\\Release\\* C:\\Deploy\\ProjectName\\ /E /Y'
        }
    }
}
    ]]>