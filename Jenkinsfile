pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/venkataramarajun/AWSRDSRestart.git'  // GitHub repository URL
        JOB_FOLDER = 'AWSRDSRB'  // Folder in Jenkins where the jobs should be created
    }

    stages {
        stage('Checkout YAML Files from GitHub') {
            steps {
                echo 'Checking out the repository from GitHub...'
                // Clone the GitHub repository containing the YAML files
                git url: "${REPO_URL}", branch: 'main'
            }
        }

        stage('List Workspace Contents') {
            steps {
                echo 'Listing workspace contents after checkout...'
                bat 'dir'  // Windows equivalent to 'ls'
            }
        }

        stage('Verify YAML Files') {
            steps {
                script {
                    echo 'Looking for YAML files...'
                    // Check the YAML files in the workspace and capture only the file names
                    def yamlFilesOutput = bat(script: 'dir /b *.yaml', returnStdout: true).trim()
                    echo "YAML files found: ${yamlFilesOutput}"

                    def yamlFiles = yamlFilesOutput.split("\n").findAll { it.endsWith('.yaml') }.collect { it.trim() }

                    if (yamlFiles.isEmpty()) {
                        error("No YAML files found to process.")
                    } else {
                        echo "Processing YAML files: ${yamlFiles}"
                        env.YAML_FILES = yamlFiles.join(',')
                    }
                }
            }
        }

        stage('Create/Update Jobs') {
            steps {
                script {
                    def yamlFiles = env.YAML_FILES.split(',')

                    echo 'Creating/Updating jobs...'
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
                                                        bat "ansible-playbook /path/to/playbook/restart_rds.yml -e db_name=${db_name} -e region=${region}"
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

        stage('Delete Jobs if YAML Removed') {
            steps {
                script {
                    echo "Checking for jobs to delete..."

                    // Use only the jobs created by the YAML files
                    def expectedJobs = env.YAML_FILES.split(',').collect { yamlFile ->
                        def yamlContent = readYaml file: "${yamlFile}"
                        return "${yamlContent.db_name}-restart-schedule"
                    }

                    // Get all the jobs in the folder (hardcoding to avoid Jenkins.getInstance())
                    def jobsInFolder = jobDsl(scriptText: """
                        folder('${JOB_FOLDER}') {
                            description('Folder for RDS Restart Jobs')
                        }
                    """)

                    // Identify jobs to delete (jobs that no longer have corresponding YAML files)
                    jobsInFolder.each { jobName ->
                        if (!expectedJobs.contains(jobName)) {
                            echo "Deleting job: ${jobName}"
                            jobDsl(scriptText: """
                                removeJob('${JOB_FOLDER}/${jobName}')
                            """)
                        }
                    }
                }
            }
        }
    }
}
