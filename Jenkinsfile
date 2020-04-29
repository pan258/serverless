    node {

        stage('Clean') {
            sh 'rm * -f -r -d -v'
        }

        stage('Clone repository') {
            checkout scm
        }

        stage('Install AWS CLI version 1') {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS_creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                sh 'curl -O https://bootstrap.pypa.io/get-pip.py'
                sh 'python get-pip.py'
                sh 'pip install aws-shell'
                sh 'aws configure set aws_access_key_id "$AWS_ACCESS_KEY_ID" | aws configure set aws_secret_access_key "$AWS_SECRET_ACCESS_KEY" | aws configure set region "us-east-1"'
            }
        }

        stage('Copy lambda_function.py') {
            sh 'mkdir -p ./lambda_jenkins_eloo_paloaltonetworks_com-master-prod19'
            sh 'mv lambda_function.py ./lambda_jenkins_eloo_paloaltonetworks_com-master-prod19/'
        }

        stage('Get serverless Defender bundle & TW_POLICY') {
            withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
                sh 'curl -k -s -u $TL_USER:$TL_PASS -X GET https://$TL_CONSOLE/api/v1/defenders/serverless/bundle?runtime=python -o twistlock_serverless_defender.zip'
                sh 'curl -k -s -u $TL_USER:$TL_PASS -X GET "https://$TL_CONSOLE/api/v1/policies/runtime/serverless/encode?consoleAddr=$TL_CONSOLE&function=lambda_jenkins_demo" -o TW_POLICY.json'
                sh 'cat TW_POLICY.json | jq -r ".data" > TW_BASE64_POLICY.json'
            }
        }

        stage('Embed function with defender') {
            unzip dir: './lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', glob: '', zipFile: 'vuln_packages.zip'
            unzip dir: './lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', glob: '', zipFile: 'twistlock_serverless_defender.zip'
            zip dir: './lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', glob: '', zipFile: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19.zip'
        }

        stage('Scan function') {
            prismaCloudScanFunction cloudFormationTemplateFile: '', functionName: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', functionPath: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19.zip', logLevel: 'info', project: '', resultsFile: 'prisma-cloud-scan-results.json'
        }

        stage('Publish scan results') {
            prismaCloudPublish resultsFilePattern: 'prisma-cloud-scan-results.json'
        }

        stage('Scan function with twistcli') {
            withCredentials([usernamePassword(credentialsId: 'twistlock_creds', passwordVariable: 'TL_PASS', usernameVariable: 'TL_USER')]) {
                sh 'curl -k -u $TL_USER:$TL_PASS --output ./twistcli https://$TL_CONSOLE/api/v1/util/twistcli'
                sh 'sudo chmod a+x ./twistcli'        
                sh './twistcli serverless scan --u $TL_USER --p $TL_PASS --address https://$TL_CONSOLE --ci --details --publish lambda_jenkins_eloo_paloaltonetworks_com-master-prod19.zip'
            } 
        }

            def TW_BASE64_POLICY = sh(script: "cat TW_BASE64_POLICY.json", returnStdout:true).trim()
            //println TW_BASE64_POLICY

        stage('Upload embedded function to AWS') {
            archiveArtifacts 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19.zip'
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS_creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
            deployLambda([alias: '', artifactLocation: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19.zip', awsAccessKeyId: "$AWS_ACCESS_KEY_ID", awsRegion: 'us-east-1', awsSecretKey: "$AWS_SECRET_ACCESS_KEY", deadLetterQueueArn: '', description: 'Function is building in Jenkins', environmentConfiguration: [configureEnvironment: true, environment: [[key: 'TW_POLICY', value: "$TW_BASE64_POLICY"]], kmsArn: ''], functionName: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', handler: 'lambda_function.handler', memorySize: '128', publish: true, role: 'arn:aws:iam::291625622176:role/lambda_basic_execution', runtime: 'python3.6', securityGroups: '', subnets: '', timeout: '30', updateMode: 'full'])
            }
        }

        stage('Invoke lambda function "/bin/ls -l"') {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS_creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                invokeLambda([awsAccessKeyId: "$AWS_ACCESS_KEY_ID", awsRegion: 'us-east-1', awsSecretKey: "$AWS_SECRET_ACCESS_KEY", functionName: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', payload: '{"body": "/bin/ls -l"}', synchronous: true])
            }
        }

        stage('Invoke lambda function "/bin/cp /etc/passwd /tmp/out"') {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'AWS_creds', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                invokeLambda([awsAccessKeyId: "$AWS_ACCESS_KEY_ID", awsRegion: 'us-east-1', awsSecretKey: "$AWS_SECRET_ACCESS_KEY", functionName: 'lambda_jenkins_eloo_paloaltonetworks_com-master-prod19', payload: '{"body": "/bin/cp /etc/passwd /tmp/out"}', synchronous: true])
            }
        }

        stage('Delete Lambda function') {
            sh 'aws lambda delete-function --function-name "lambda_jenkins_eloo_paloaltonetworks_com-master-prod19"'
        }

        stage('Clean') {
            sh 'rm * -f -r -d -v'
        }
    }
