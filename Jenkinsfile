pipeline {
  agent any
  parameters {
    string(name: 'ApplicationScope', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy.')
    string(name: 'ApplicationScopeWithTests', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy (including test applications)')
    string(name: 'TriggeredBy', defaultValue: 'N/A', description: 'Name of LifeTime user that triggered the pipeline remotely.')
  }
  options { skipStagesAfterUnstable() }
  environment {
    // Artifacts Folder
    ArtifactsFolder = "Artifacts"
    // Trigger Manifest Specific Variables
    ManifestFolder = "trigger_manifest"
    ManifestFile = "trigger_manifest.json"
    // LifeTime Specific Variables
    LifeTimeHostname = 'https://sysmanager-dev.outsystemscloud.com'
    LifeTimeAPIVersion = 2
    // Authentication Specific Variables
    AuthorizationToken = 'eyJhbGciOiJSUzI1NiIsImtpZCI6ImQ0YTEzOTMwLTM1ZGItNDQwMi1iN2EyLWVmZWVlN2FmOGIyOCIsInR5cCI6IkpXVCJ9.eyJpYXQiOiIxNzAwMjM1MDIwIiwic3ViIjoiTTJVeVpqRXlZMkl0TVdVeFlTMDBZamMyTFRrMU5tRXRZMlEwTVdRNVlUTmtOR1F4IiwianRpIjoidWFBVkZySUJCcCIsImV4cCI6MTc2MzM5MzQyMCwiaXNzIjoibGlmZXRpbWUiLCJhdWQiOiJsaWZldGltZSJ9.kTDq9ktVNh1BTt4znqP3AcyvQdFPbokOq3d6xSR2PUX8DFC5Aw7NhwI9wDH3XDOoJbBVYdDmLtBZlw1CKkokHMUjCf6WC0gpuVPAKwYUbasEeVl8uXcc3cHmfUYxlwmbcTnIfZ3KeeAd3vRlhFHUDs3YH7OIHdmwqT8rlKpoZA_y7pPcJeBCJDtT-AEDuy0GPJSUSr1ZHiH_xxIdCAeFj2sy1o-d4VLOPdNS6OJ6lCpWXoSkQXG8l5Cow5Q3Zr7gIQIP4AxF4PNT6ClyU5-62a7eKliSH1DMogb1mDVgg5hxLhn1Br-VhV_nKDuD44JIGVwc27kfsDVR-fcMpf5VVw' //credentials('LifeTimeServiceAccountToken')
    // Environments Specification Variables

    DevelopmentEnvironment = 'Development'
    ProductionEnvironment = 'Production'
    
    DevelopmentEnvironmentLabel = 'Development'
    //RegressionEnvironmentLabel = 'Regression'
    //AcceptanceEnvironmentLabel = 'Acceptance'
    //PreProductionEnvironmentLabel = 'PreProduction'
    //ProductionEnvironmentLabel = 'Production'
    // Regression URL Specification
    CICDProbeEnvironmentURL = 'https://sysmanager-dev.outsystemscloud.com'
    BDDFrameworkEnvironmentURL = 'https://sysmanager-dev.outsystemscloud.com'
    // OutSystems PyPI package version
    OSPackageVersion = '0.6.0'
  }
  stages {
    stage('Install Python Dependencies') {
      steps {
        sh(script: "mkdir ${env.ArtifactsFolder}", label: 'Create artifacts folder')
        sh(script: 'pip3 install -q -I virtualenv --user', label: 'Install Python virtual environments')
        withPythonEnv(pythonInstallation: 'python3') {
          sh(script: "pip3 install -U outsystems-pipeline==\"${env.OSPackageVersion}\"", label: 'Install required packages')
        }

      }
    }

    stage('Get and Deploy Latest Tags') {
      steps {
        withPythonEnv(pythonInstallation: 'python3') {

          echo "OLA 1"
          echo "Auth token: '${env.AuthorizationToken}'"
          
          echo "Pipeline run triggered remotely by '${params.TriggeredBy}' for the following applications (including tests): '${params.ApplicationScopeWithTests}'"
          sh(script: "python3 -m outsystems.pipeline.fetch_lifetime_data --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion}", label: 'Retrieve list of Environments and Applications')
          lock(resource: 'deployment-plan-REG') {
            sh(script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.DevelopmentEnvironment}\" --destination_env \"${env.ProductionEnvironment}\" --app_list \"${params.ApplicationScopeWithTests}\"", label: "Deploy latest application tags (including tests) to ${env.RegressionEnvironment}")
          }

        }

      }
    }

    stage('Run Regression') {
      when {
        expression {
          return params.ApplicationScope != params.ApplicationScopeWithTests
        }

      }
      post {
        always {
          junit(testResults: "${env.ArtifactsFolder}/junit-result.xml", allowEmptyResults: true, skipPublishingChecks: true)
        }

      }
      steps {
        withPythonEnv(pythonInstallation: 'python3') {
          sh(script: "python3 -m outsystems.pipeline.generate_unit_testing_assembly --artifacts \"${env.ArtifactsFolder}\" --app_list \"${params.ApplicationScopeWithTests}\" --cicd_probe_env ${env.ProbeEnvironmentURL} --bdd_framework_env ${env.BddEnvironmentURL}", label: 'Generate URL endpoints for BDD test suites')
          sh(script: "python3 -m outsystems.pipeline.evaluate_test_results --artifacts \"${env.ArtifactsFolder}\"", returnStatus: true, label: 'Run BDD test suites and generate JUnit test report')
        }

      }
    }

    stage('Deploy Acceptance') {
      steps {
        withPythonEnv(pythonInstallation: 'python3') {
          lock(resource: 'deployment-plan-ACC') {
            sh(script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.RegressionEnvironment}\" --destination_env \"${env.AcceptanceEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}/deployment_data/deployment_manifest.cache\"", label: "Deploy latest application tags to ${env.AcceptanceEnvironment}")
          }

        }

      }
    }

    stage('Accept Changes') {
      steps {
        milestone(ordinal: 40, label: 'before-approval')
        timeout(time: 1, unit: 'DAYS') {
          input 'Accept changes and deploy to Production?'
        }

        milestone(ordinal: 50, label: 'after-approval')
      }
    }

    stage('Deploy Dry-Run') {
      steps {
        withPythonEnv(pythonInstallation: 'python3') {
          lock(resource: 'deployment-plan-PRE') {
            sh(script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.AcceptanceEnvironment}\" --destination_env \"${env.PreProductionEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}/deployment_data/deployment_manifest.cache\"", label: "Deploy latest application tags to ${env.PreProductionEnvironment}")
          }

        }

      }
    }

    stage('Deploy Production') {
      steps {
        withPythonEnv(pythonInstallation: 'python3') {
          lock(resource: 'deployment-plan-PRD') {
            sh(script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.PreProductionEnvironment}\" --destination_env \"${env.ProductionEnvironment}\" --app_list \"${params.ApplicationScope}\" --manifest \"${env.ArtifactsFolder}/deployment_data/deployment_manifest.cache\"", label: "Deploy latest application tags to ${env.ProductionEnvironment}")
          }

        }

      }
    }

  }

  post {
    always {
      dir("${env.ArtifactsFolder}") {
        archiveArtifacts '**/*.cache'
      }

    }

    failure {
      dir("${env.ArtifactsFolder}") {
        archiveArtifacts(artifacts: 'DeploymentConflicts', allowEmptyArchive: true)
      }

    }

    cleanup {
      dir("${env.ArtifactsFolder}") {
        deleteDir()
      }

    }

  }
}
