pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/venkataramarajun/AWSRDSRestart.git'  // GitHub repository URL
        YAML_DIR = 'AWSRDSRestart'  // Directory where your YAML files are stored in the repo
        JOB_FOLDER = 'AWSRDSRB'  // Folder in Jenkins where the jobs should be created
    }

    stages {
        stage('Checkout YAML Files') {
            steps {
                echo 'Checking out the repository...'
                // Clone the GitHub repository containing the YAML files
                git url: "${REPO_URL}", branch: 'main'
            }
        }

        stage('Verify Workspace') {
            steps {
                echo 'Listing workspace contents...'
                // List the files in the workspace to ensure the files are there
                sh 'ls -la'
            }
        }

        stage('Verify YAML Files') {
            steps {
                script {
                    echo 'Looking for YAML files...'
                    // Check the YAML files in the workspace
                    def yamlFilesOutput = sh(script: "ls *.yaml || echo 'No YAML files found'", returnStdout: true).trim()
                    echo "YAML files found: ${yamlFilesOutput}"

                    def yamlFiles = yamlFilesOutput.split("\n").findAll { it.endsWith('.yaml') }

                    if (yamlFiles.isEmpty()) {
                        error("No YAML files found to process.")
                    } else {
                        echo "Processing YAML files: ${yamlFiles}"
                    }
                }
            }
        }

        stage('Create Folder and Process YAML Files') {
            steps {
                script {
                    echo 'Creating job folder...'
                    // Create folder if it doesn't exist
                    jobDsl scriptText: """
                    folder('${JOB_FOLDER}') {
                        description('Folder for RDS Restart Jobs')
                    }
                    """

                    // For each YAML file, read the content and create/update jobs
                    yamlFiles.each { yamlFile ->
                        echo "Processing ${yamlFile}..."
                        def yamlContent = readYaml file: "${yamlFile}"
                        def db_name = yamlContent.db_name
                        def region = yamlContent.region
                        def schedule = yamlContent.schedule

                        echo "Creating job for DB: ${db_name}, Region: ${region}, Schedule: ${schedule}"

                        // Create or update the Jenkins job using Job DSL
                        jobDsl scriptText: """
                        pipelineJob('${JOB_FOLDER}/${db_name}-restart-schedule') {
                            description('Job to restart RDS DB: ${db_name}')
                            triggers {
                                cron('${schedule}')
                            }
                            definition {
                                cps {
                                    script('''
                                        pipeline {
                                            agent any
                                            stages {
                                                stage('Restart RDS DB') {
                                                    steps {
                                                        echo "Restarting RDS DB: ${db_name} in ${region}"
                                                        sh "ansible-playbook /path/to/playbook/restart_rds.yml -e db_name=${db_name} -e region=${region}"
                                                    }
                                                }
                                            }
                                        }
                                    ''')
                                }
                            }
                        }
                        """
                    }
                }
            }
        }
    }
}
