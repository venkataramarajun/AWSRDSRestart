pipeline {
    agent any

    environment {
        REPO_URL = 'https://github.com/venkataramarajun/AWSRDSRestart.git'  // GitHub repository URL
        JOB_FOLDER = 'AWSRDSRB_final'  // Folder in Jenkins where the jobs should be created
    }

    stages {
        stage('Checkout YAML Files from GitHub') {
            steps {
                echo 'Checking out the repository from GitHub...'
                git url: "${REPO_URL}", branch: 'main'  // Clone the repository
            }
        }

        stage('List Workspace Contents') {
            steps {
                echo 'Listing workspace contents after checkout...'
                script {
                    def yamlFiles = findFiles(glob: '**/*.yaml')  // Find all .yaml files in the workspace
                    if (yamlFiles.length == 0) {
                        error("No YAML files found to process.")
                    } else {
                        echo "YAML files found: ${yamlFiles.collect { it.path }}"  // Collect file paths from FileWrapper objects
                        env.YAML_FILES = yamlFiles.collect { it.path }.join(',')
                    }
                }
            }
        }

        stage('Create Job Folder') {
            steps {
                script {
                    // Create the job folder if it doesn't exist using Job DSL
                    jobDsl scriptText: """
                    folder('${JOB_FOLDER}') {
                        displayName('${JOB_FOLDER}')
                        description('Folder for RDS restart jobs')
                    }
                    """
                    echo "Job folder ${JOB_FOLDER} created or already exists."
                }
            }
        }

        stage('Verify and Process YAML Files') {
            steps {
                script {
                    def yamlFiles = env.YAML_FILES.split(',')
                    def jobNamesCreated = []

                    yamlFiles.each { yamlFile ->
                        echo "Processing ${yamlFile}..."
                        def yamlContent = readYaml file: "${yamlFile}"

                        def db_name = yamlContent.db_name
                        def region = yamlContent.region
                        def schedule = yamlContent.schedule

                        if (!db_name || !region || !schedule) {
                            echo "Skipping ${yamlFile} due to missing required fields (db_name, region, schedule)."
                        } else {
                            echo "Creating or Updating job for DB: ${db_name}, Region: ${region}, Schedule: ${schedule}"

                            // Create or update the Jenkins job using Job DSL with cron schedule
                            jobDsl scriptText: """
                            pipelineJob('${JOB_FOLDER}/${db_name}-restart-schedule') {
                                description('Job to restart RDS DB: ${db_name}')
                                triggers {
                                    cron('${schedule}')  // Set the cron schedule from YAML
                                }
                                definition {
                                    cps {
                                        script('''
                                            pipeline {
                                                agent any
                                                stages {
                                                    stage('Restart RDS DB') {
                                                        steps {
                                                            echo "Restarting RDS DB: ${db_name} in region: ${region}"
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
                            jobNamesCreated << "${db_name}-restart-schedule"  // Track created or updated jobs
                        }
                    }

                    // Set the environment variable with the created job names
                    env.CREATED_JOBS = jobNamesCreated.join(',')
                }
            }
        }

        stage('Delete Jobs if YAML Removed') {
            steps {
                script {
                    // Get jobs created in this run
                    def createdJobs = env.CREATED_JOBS.split(',')
                    
                    // List all existing jobs in the folder
                    def existingJobs = Jenkins.instance.getItemByFullName("${JOB_FOLDER}").items.collect { it.name }

                    echo "Existing jobs in Jenkins: ${existingJobs}"
                    echo "Created/Updated jobs in this run: ${createdJobs}"

                    // Find jobs that are in Jenkins but not in the YAML files (i.e., jobs to delete)
                    def jobsToDelete = existingJobs - createdJobs

                    jobsToDelete.each { jobName ->
                        echo "Deleting job: ${jobName} as it no longer exists in YAML files."
                        try {
                            // Delete the job if it exists in Jenkins
                            Jenkins.instance.getItemByFullName("${JOB_FOLDER}/${jobName}")?.delete()
                            echo "Deleted job: ${jobName}"
                        } catch (Exception e) {
                            echo "Failed to delete job: ${jobName}. It may already be removed."
                        }
                    }
                }
            }
        }
    }
}
