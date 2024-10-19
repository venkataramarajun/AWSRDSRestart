pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/venkataramarajun/AWSRDSRestart.git'  // GitHub repository URL
        JOB_FOLDER = 'AWSRDSRB'  // Folder in Jenkins where the jobs should be created
    }

    stages {
        stage('Checkout YAML Files') {
            steps {
                // Clone the GitHub repository containing the YAML files with credentials if needed
                git url: "${REPO_URL}", branch: 'main', credentialsId: 'your-credentials-id'
            }
        }

        stage('Create Folder and Process YAML Files') {
            steps {
                script {
                    // Create folder if it doesn't exist
                    jobDsl scriptText: """
                    folder('${JOB_FOLDER}') {
                        description('Folder for RDS Restart Jobs')
                    }
                    """

                    // List all YAML files in the root directory (adjust if YAML files are in a subfolder)
                    def yamlFiles = sh(script: "ls *.yaml", returnStdout: true).trim().split("\n")

                    // For each YAML file, read the content and create/update jobs
                    yamlFiles.each { yamlFile ->
                        def yamlContent = readYaml file: "${yamlFile}"
                        def db_name = yamlContent.db_name
                        def region = yamlContent.region
                        def schedule = yamlContent.schedule

                        echo "Processing job for DB: ${db_name}, Region: ${region}, Schedule: ${schedule}"

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
