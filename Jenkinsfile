/*
 * Jenkins pipeline to generate client-specific Jenkinsfiles
 * and deploy them as executable jobs in Jenkins.
 * 
 * Features:
 * - Sets build name = CLIENT_NAME and adds job URL in description.
 * - Supports config/template-driven Jenkinsfile generation.
 * - Deploys generated pipeline to client repository and registers job in Jenkins.
 * 
 * Requirements:
 * - GitHub credentials (HTTPS or SSH).
 * - Jenkins API credentials for job creation and triggering.
 * - Shared library: jenkins-xaas-shared-lib.
 */


@Library('jenkins-xaas-shared-lib') _

pipeline {
    agent any

    // Input Parameters
    parameters {
        // Required parameters
        validatingString(
            name: 'CLIENT_NAME',
            defaultValue: '',
            regex: /^[\S ].*$/, // Must start with a non-space character
            failedValidationMessage: 'CLIENT_NAME cannot be empty',
            description: 'Unique client identifier (used in config path and job name)'
        )
        validatingString(
            name: 'REPO_URL',
            defaultValue: '',
            regex: /^[\S ].*$/, // Must start with a non-space character
            failedValidationMessage: 'REPO_URL cannot be empty',
            description: 'Target client repo URL.'
        )

        // Configuration and templates parameters
        string(
            name: 'CONFIG_FILE',
            defaultValue: 'configs/sample_config.yaml',
            description: 'Path to configuration file (.env or .yaml)'
        )
        choice(
            name: 'TEMPLATE_NAME',
            choices: ['basic', 'docker-node'],
            description: 'Template to use from pipeline-generator/templates/'
        )
        string(
            name: 'OUTPUT_FILE',
            defaultValue: 'output/Jenkinsfile',
            description: 'Path where generated Jenkinsfile will be written'
        )

        // Optional overrides
        string(name: 'BRANCH_NAME', defaultValue: '', description: 'Branch override (optional)')
        string(name: 'BUILD_TOOL', defaultValue: '', description: 'Build tool override (optional)')
        string(name: 'BUILD_STEPS', defaultValue: '', description: 'Custom build steps (optional)')

        // Flags
        booleanParam(
            name: 'PREVIEW_MODE',
            defaultValue: false,
            description: 'Preview only (no file will be generated)'
        )

        booleanParam(
            name: 'VALIDATE_ONLY',
            defaultValue: false,
            description: '''
                Validate inputs only (no generation). <br/>
                <span style="color:red; font-weight:bold;">
                NOTE: When enabled, if build fails, neither a preview will be shown nor an output file will be generated.
                The build will be marked as unstable.
                </span>
            '''
        )

        booleanParam(
            name: 'STRICT_MODE',
            defaultValue: true,
            description: '''
                Enforce strict validation rules. <br/>
                <span style="color:red; font-weight:bold;">
                NOTE: When enabled, the build will fail on validation errors.
                </span>
            '''
        )
    }

    // Environment variables
    environment {
        // GitHub credentials:
        // - HTTPS token: used for cloning repos or deploying Jenkinsfiles to client repos.
        // - SSH token:   typically used for cloning repos.
        GITHUB_HTTPS_TOKEN        = "github-https-token"
        GITHUB_SSH_TOKEN          = "github-ssh-creds"

        // Jenkins API credentials for job creation & triggering
        JENKINS_API_TOKEN         = "jenkins-api-token"

        // Path to Jenkinsfile generator inside checked-out repo
        JENKINSFILE_GENERATOR_DIR = "${WORKSPACE}/jenkins-xaas/pipeline-generator"

        // Configure branch of forked repository
        FORKED_BRANCH = "main"

        // Configure branch of client repo to deploy
        TARGET_BRANCH = "main"
    }

    // Pipeline Stages
    stages {
        stage('Initialize') {
            steps {
                cleanWs()
                script {
                    // Set build metadata
                    currentBuild.displayName = "${params.CLIENT_NAME}"
                    currentBuild.description = "Generating Jenkinsfile for ${params.CLIENT_NAME}"
                }
            }
        }

        stage('Checkout Generator Repository') {
            steps {
                script {
                    /*
                    * Clone jenkins-xaas repo (contains generator + templates + job deployer).
                    * IMPORTANT:
                    * - The repo URL and credentials must match in style:
                    *   → If using SSH URL (`git@github.com:...`), use SSH credentials.
                    *   → If using HTTPS URL (`https://github.com/...`), use HTTPS token.
                    * - Mixing styles (SSH URL with HTTPS creds or vice versa) will fail.
                    */
                    cloneRepo(
                        url: '', // Configure forked repo URL
                        branch: "${FORKED_BRANCH}",
                        credsId: "", // Configure credsId according to URL style
                        targetDir: 'jenkins-xaas'
                    )
                }
            }
        }

        stage('Generate Jenkinsfile') {
            steps {
                script {
                    // Run generator with provided params
                    generateJenkinsfile(
                        configFile: params.CONFIG_FILE,
                        outputFile: params.OUTPUT_FILE,
                        templateName: params.TEMPLATE_NAME,
                        branchName: params.BRANCH_NAME,
                        buildTool: params.BUILD_TOOL,
                        buildSteps: params.BUILD_STEPS,
                        repoUrl: params.REPO_URL,
                        previewMode: params.PREVIEW_MODE,
                        validateOnly: params.VALIDATE_ONLY,
                        strictMode: params.STRICT_MODE
                    )
                }
            }
            // Post Actions
            post {
                success {
                    echo("Jenkinsfile generated successfully.")
                }
                failure {
                    echo("Jenkinsfile generation failed. Check logs for details.")
                }
                unstable {
                    echo("Generate Jenkinsfile stage unstable. Check logs for details.")
                }
            }
        }

        stage('Deploy Jenkinsfile to GitHub') {
            when {
                expression { 
                    !(params.VALIDATE_ONLY || params.PREVIEW_MODE || currentBuild.result == 'UNSTABLE')
                }
            }
            steps {
                // Commit & push generated Jenkinsfile to client repo
                deployJenkinsfile(
                    repoUrl: params.REPO_URL,
                    clientName: params.CLIENT_NAME,
                    sourceFile: "${JENKINSFILE_GENERATOR_DIR}/${params.OUTPUT_FILE}",
                    targetBranch: "${TARGET_BRANCH}"
                )
            }
            // Post Actions
            post {
                success {
                    echo("Jenkinsfile deployed successfully.")
                }
                failure {
                    echo("Jenkinsfile deployment failed. Check logs for details.")
                }
            }
        }

        stage('Deploy Client Job & Trigger Build') {
            when {
                expression { 
                    !(params.VALIDATE_ONLY || params.PREVIEW_MODE || currentBuild.result == 'UNSTABLE')
                }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${JENKINS_API_TOKEN}",
                        usernameVariable: 'JENKINS_USER',
                        passwordVariable: 'JENKINS_API_TOKEN'
                    )
                ]) {
                    // Create/Update Jenkins job for client and trigger initial build
                    deployClientJob(
                        clientName: params.CLIENT_NAME,
                        repoUrl: params.REPO_URL
                    )

                    script {
                        // Add clickable job URL to build description
                        def jobUrl = "${env.JENKINS_URL}job/${params.CLIENT_NAME}-pipeline/"
                        def pipelineUrl = """${params.REPO_URL}/blob/${TARGET_BRANCH}/${params.CLIENT_NAME}/Jenkinsfile"""
                        currentBuild.description = """
                            ${params.CLIENT_NAME} Job Jenkinsfile: <a href="${pipelineUrl}">${pipelineUrl}</a><br/>
                            ${params.CLIENT_NAME} Job URL: <a href="${jobUrl}">${jobUrl}</a>
                        """
                    }
                }
            }
            // Post Actions
            post {
                success {
                    echo("Job created and triggered successfully.")
                }
                failure {
                    echo("Job creation failed. Check logs for details.")
                }
            }
        }
    }
    post {
        success {
            echo("Pipeline completed successfully for ${params.CLIENT_NAME}.")
        }
        failure {
            echo("Pipeline failed for ${params.CLIENT_NAME}. Check logs.")
        }
        unstable {
            echo("Pipeline unstable. Check logs.")
        }
        aborted {
            echo("Pipeline aborted.")
        }
    }
}
