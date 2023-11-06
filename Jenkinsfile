pipeline {
  agent any
  parameters {
    // Pipeline parameters are automatically filled by LT Trigger plugin
    string(name: 'TriggerManifest', defaultValue: '', description: 'Trigger manifest artifact (in JSON format) for the current pipeline run.')
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
    LifeTimeHostname = 'sysmanager-dev.outsystemscloud.com'
    LifeTimeAPIVersion = 2
    // Authentication Specific Variables
    AuthorizationToken = 'eyJhbGciOiJSUzI1NiIsImtpZCI6ImQ0YTEzOTMwLTM1ZGItNDQwMi1iN2EyLWVmZWVlN2FmOGIyOCIsInR5cCI6IkpXVCJ9.eyJpYXQiOiIxNjk4MjQ2ODI5Iiwic3ViIjoiTTJVeVpqRXlZMkl0TVdVeFlTMDBZamMyTFRrMU5tRXRZMlEwTVdRNVlUTmtOR1F4IiwianRpIjoiWVVYaTNreXRYayIsImV4cCI6MTc2MTQwNTIyOSwiaXNzIjoibGlmZXRpbWUiLCJhdWQiOiJsaWZldGltZSJ9.fpQNuWMXoGZ_8z8ub-vO6pKJccyxE1iSlqJAisBvS_XO7-X9i6N19EdviXU-D0uXJ4QET8pKsxG18rZ9tigRmd8j4xA5uyA2KQM0t8VEwjvx-wkv1bwhYEADHts4HkhTczRNjfZa_ASaMlf3NUdmRSlfUrAbMN8oICmtzODgxMulz_UAWvBNI8jzBezJHn1hng0u6nFB2Ts7VUaoEedYPahrrIYzhlDmjOJ70NPd-OSBUcONaY7_QdUUe2_SKSBFakMSmaJuHGDqUil9MdE8Nosg4KJSwBo-pwQ8_DvRrL4EU_U_2MrFy3U4zQRuvUZLazSNeZrFjh4C3e_p1jZdrg' //credentials('LifeTimeServiceAccountToken')
    // Environments Specification Variables
    
    * Pipeline for 5 Environments:
    *   Development Environment    -> Where you develop your applications. This should be the default environment you connect with service studio.
    *   Regression Environment     -> Where your automated tests will test your applications.
    *   Acceptance Environment     -> Where you run your acceptance tests of your applications.
    *   PreProduction Environment  -> Where you prepare your apps to go live.
    *   Production Environment     -> Where your apps are live.
    */
    DevelopmentEnvironmentLabel = 'Development'
    RegressionEnvironmentLabel = 'Regression'
    AcceptanceEnvironmentLabel = 'Acceptance'
    PreProductionEnvironmentLabel = 'PreProduction'
    ProductionEnvironmentLabel = 'Production'
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
            sh(script: "python3 -m outsystems.pipeline.deploy_latest_tags_to_target_env --artifacts \"${env.ArtifactsFolder}\" --lt_url ${env.LifeTimeHostname} --lt_token ${env.AuthorizationToken} --lt_api_version ${env.LifeTimeAPIVersion} --source_env \"${env.DevelopmentEnvironment}\" --destination_env \"${env.RegressionEnvironment}\" --app_list \"${params.ApplicationScopeWithTests}\"", label: "Deploy latest application tags (including tests) to ${env.RegressionEnvironment}")
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
  environment {
    ArtifactsFolder = 'Artifacts'
    LifeTimeHostname = 'https://sysmanager-dev.outsystemscloud.com'
    LifeTimeAPIVersion = 2
    DevelopmentEnvironment = 'Development'
    ProductionEnvironment = 'Production'
    OSPackageVersion = '0.4.0'
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
  options {
    skipStagesAfterUnstable()
  }
  parameters {
    string(name: 'ApplicationScope', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy.')
    string(name: 'ApplicationScopeWithTests', defaultValue: '', description: 'Comma-separated list of LifeTime applications to deploy (including test applications)')
    string(name: 'TriggeredBy', defaultValue: 'N/A', description: 'Name of LifeTime user that triggered the pipeline remotely.')
  }
}
