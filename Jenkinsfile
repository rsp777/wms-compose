// Jenkinsfile
pipeline {
    agent any

    environment {
        // Repository and Docker stack info
        REPO_URL      = 'https://github.com/rsp777/wms-compose.git'
        BRANCH_NAME   = 'main'  // Default branch for checkout
        OTHER_BRANCH  = 'main'  // Replace with the actual different branch where your file is located
        ENDPOINT_ID   = 7
        BUILD_FOLDER  = '/var/lib/jenkins/workspace/Host-Integration-Services-Deployment/item-integration-service-deployment/host-integration-services/item-integration-service'  // Updated based on your description
        PORTAINER_URL = 'http://192.168.29.137:9000/api'  // Base URL
        SWARM_ENDPOINT_ID = 'pgp82yiq9iqg2gp8h28cug3te'  // Docker Swarm endpoint ID
        STACK_NAME    = 'item_integration_service'
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    def repoUrl = env.REPO_URL
                    def desiredBranch = (env.BRANCH_NAME != env.OTHER_BRANCH) ? env.OTHER_BRANCH : env.BRANCH_NAME
                    
                    // Check if the workspace is a Git repo and matches the URL and branch
                    def isGitRepo = sh(script: 'git rev-parse --is-inside-work-tree 2>/dev/null && echo "true" || echo "false"', returnStdout: true).trim()
                    
                    if (isGitRepo == 'true') {
                        def currentRemoteUrl = sh(script: 'git config --get remote.origin.url', returnStdout: true).trim()
                        def currentBranch = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
                        
                        if (currentRemoteUrl == repoUrl && currentBranch == desiredBranch) {
                            echo "Repository and branch already match; skipping checkout."
                        } else {
                            echo "Repository or branch mismatch; performing checkout."
                            checkout scm: [$class: 'GitSCM', branches: [[name: desiredBranch]], userRemoteConfigs: [[url: repoUrl]]]
                        }
                    } else {
                        echo "Workspace is not a Git repo; performing checkout."
                        checkout scm: [$class: 'GitSCM', branches: [[name: desiredBranch]], userRemoteConfigs: [[url: repoUrl]]]
                    }
                }
            }
        }

        stage('Authenticate with Portainer') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'PORTAINER_CRED', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        try {
                            echo "Attempting to authenticate with Portainer..."
                            def jsonPayload = "{\"Username\":\"${USERNAME}\",\"Password\":\"${PASSWORD}\"}"
                            echo "Sending JSON payload: ${jsonPayload}"  // Debug log to verify the payload
                            
                            def tokenResponse = sh(
                                script: """curl -s -X POST "${PORTAINER_URL}/auth" \\
                                    -H "Content-Type: application/json" \\
                                    -d '${jsonPayload}' """,  // Use single-quoted string for the payload to avoid issues
                                returnStdout: true
                            ).trim()
                            //echo "Raw token response: ${tokenResponse}"  // Debug log
                            
                            def jsonSlurper = new groovy.json.JsonSlurper()
                            def jsonResponse = jsonSlurper.parseText(tokenResponse)
                            env.PORTAINER_TOKEN = jsonResponse.jwt  // Extract the JWT token
                            //echo "Retrieved token: ${env.PORTAINER_TOKEN}"  // Debug log
                            if (!env.PORTAINER_TOKEN) {
                                error "No JWT token retrieved. Authentication may have failed."
                            }
                        } catch (Exception e) {
                            error "Authentication failed: ${e.message}. Check credentials and API response."
                        }
                    }
                    echo "✅ Portainer API token retrieved successfully."
                }
            }
        }

        stage('Deploy Stack') {
    steps {
        script {
            try {
                echo "Deploying stack : ${env.STACK_NAME}"
                def boundary = "----WebKitFormBoundary${UUID.randomUUID().toString().replace('-', '')}"
                def url = new URL("${PORTAINER_URL}/stacks/create/swarm/file?endpointId=${ENDPOINT_ID}")
                def connection = url.openConnection()
                connection.setRequestMethod("POST")
                connection.setDoOutput(true)
                connection.setRequestProperty("Authorization", "Bearer ${PORTAINER_TOKEN}")
                connection.setRequestProperty("Content-Type", "multipart/form-data; boundary=${boundary}")

                def outputStream = connection.getOutputStream()
                def writer = new PrintWriter(new OutputStreamWriter(outputStream, "UTF-8"), true)

                // Helper to write form fields
                def writeFormField = { name, value ->
                    writer.println("--${boundary}")
                    writer.println("Content-Disposition: form-data; name=\"${name}\"")
                    writer.println()
                    writer.println(value)
                }

                // Helper to write file field
                def writeFileField = { name, filePath ->
                    def file = new File(filePath)
                    writer.println("--${boundary}")
                    writer.println("Content-Disposition: form-data; name=\"${name}\"; filename=\"${file.name}\"")
                    writer.println("Content-Type: application/octet-stream")
                    writer.println()
                    writer.flush()
                    outputStream.write(file.bytes)
                    outputStream.flush()
                    writer.println()
                }

                // Write form fields
                writeFormField("Name", STACK_NAME)
                writeFormField("SwarmID", SWARM_ENDPOINT_ID)
                writeFormField("Env", "[]")
                writeFileField("file", "${BUILD_FOLDER}/compose.yaml")

                // End boundary
                writer.println("--${boundary}--")
                writer.close()
               
                // Read response
                def responseCode = connection.getResponseCode()
                def responseMessage = connection.getInputStream().getText()

                if (responseCode == 200 || responseCode == 201) {
                    echo "✅ Stack deployed successfully. Response: ${responseMessage}"
                } else {
                    error "❌ Deployment failed. HTTP ${responseCode}: ${responseMessage}"
                }

            } catch (Exception e) {
                // Enhanced error handling: Print exact error and stack trace
                echo "Exact error: ${e.toString()}"  // This includes the full exception details
                def sw = new StringWriter()
                def pw = new PrintWriter(sw)
                e.printStackTrace(pw)  // Capture the stack trace
                echo "Stack trace:\n${sw.toString()}"  // Output the stack trace to the console
                error "❌ Deployment failed due to error: ${e.message}. Please verify the JWT token and endpoint. Full details and stack trace are logged above."
            }
        }
    }
}

    }

    post {
        always { echo 'Pipeline completed.' }
        success { echo 'Deployment successfully executed!' }
        failure { echo 'Deployment failed, check logs.' }
    }
}
