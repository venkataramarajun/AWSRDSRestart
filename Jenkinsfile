    pipeline {
        agent any
    
        environment {
            REPO_URL = 'https://github.com/venkataramarajun/AWSRDSRestart.git'  // GitHub repository URL
            JOB_FOLDER = 'AWSRDSRB_final'  // Folder in Jenkins where the jobs should be created
            JOB_TRACKING_FILE = 'created_jobs.log'  // File to track created jobs
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
                                echo "Creating job for DB: ${db_name}, Region: ${region}, Schedule: ${schedule}"
    
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
                                jobNamesCreated << "${db_name}-restart-schedule"  // Track created jobs
                            }
                        }
    
                        writeFile file: "${JOB_TRACKING_FILE}", text: jobNamesCreated.join('\n')  // Store created jobs in file
                        env.CREATED_JOBS = jobNamesCreated.join(',')
                    }
                }
            }
    
            stage('Delete Jobs if YAML Removed') {
                steps {
                    script {
                        def createdJobs = env.CREATED_JOBS.split(',')  // Get jobs created in this run
    
                        // List the jobs tracked from the last run
                        def previousJobs = readFile("${JOB_TRACKING_FILE}").split('\n').findAll { it }
    
                        echo "Previous jobs: ${previousJobs}"
                        echo "Created jobs in this run: ${createdJobs}"
    
                        // Jobs to delete are those in the previous list but not in the current run
                        def jobsToDelete = previousJobs - createdJobs
    
                        jobsToDelete.each { jobName ->
                            echo "Deleting job: ${jobName} as it no longer exists in YAML files."
                            jobDsl removedJobAction: 'DELETE', scriptText: """
                                pipelineJob('${JOB_FOLDER}/${jobName}') {
                                    // This job will be deleted
                                }
                            """
                        }
                    }
                }
            }
        }
    }
