import groovy.json.JsonOutput
pipeline {
    agent {
        node {
            label 'dl'
        }
    }
    environment {
        _version = sh(script: "echo `date '+%Y-%m-%d_%H:%M:%S'`", returnStdout: true).trim()
        dev_tag="${branch_name}"
        RESULT_DIR = "${WORKSPACE}/../bot_slam_test_results" 
    }

    stages {
        stage('Pull code') {
            steps {
                echo 'Pulling code...'
                script {
                    checkout([$class: 'GitSCM', branches: [[name: '${branch_name}']], extensions: [[$class: 'CloneOption', noTags: false, reference: '', shallow: false, timeout: 120], [$class: 'PruneStaleBranch'], [$class: 'CheckoutOption', timeout: 120], [$class: 'GitLFSPull'], [$class: 'CleanCheckout', deleteUntrackedNestedRepositories: true]], userRemoteConfigs: [[credentialsId: '9d477d64-a237-46a5-8fb6-8df89c0899f2', url: 'git@github.com:TrifoRobotics/bot.git']]])
                }
            }
        }
        stage('init code') {
            environment {
                GIT_SSH_CRED = credentials('TrifoRobotics-core')
            }
            steps {
                dir('./base') {
                    sh '''
                        export GIT_SSH_COMMAND=\"ssh -i ${GIT_SSH_CRED}\"
                        common_status=\$(git submodule status common)
                        git submodule update --init --force common
                       
                        git clean -xdf
                        git submodule update --init --force core
                        cd ./core && git submodule update --init --force algo/common || true
                    '''
                }
            }
            
        }
        stage('sync data') {
            environment {
                AWS_CONFIG_DIR = "${WORKSPACE}/../sync_bot_slam_data/awsconfig"
                AWS_DIR = 's3://trifo-auto-test/path/jenkins_trifo_CI/test_data'
                SYNC_AWS_DIR = "${WORKSPACE}/../sync_bot_slam_data/test_data"
                SYNC_WORKSPACE = "${WORKSPACE}/../sync_bot_slam_data"
                START_TIME = sh(script: "echo `date '+%Y-%m-%d_%H:%M:%S'`", returnStdout: true).trim()
            }
            when {
                expression {
                    echo "$env.AwsSys"
                    if (env.AwsSys == "false") {
                        echo "close sync data"
                        env.SYNC_STATUS = '0'
                        env.SYNC_RESULT='CLOSED'
                        env.SYNC_DURATION='0'

                        // Create a folder to store the data of this build
                        env.CURRENT_RESULT_DIR="${env.RESULT_DIR}/results_${_version}"
                        sh "mkdir ${env.CURRENT_RESULT_DIR}"
                        build job: 'bot_data_test_results', parameters: [string(name: 'TODAY_RESULT_DIR', value: "results_${_version}")]
                        echo "The folder path created this time:$env.CURRENT_RESULT_DIR"
                    }
                    return (env.AwsSys == "true")
                }
            }
            steps {
                dir("${WORKSPACE}/../sync_bot_slam_data") {
                    script {
                        echo "SYNC_AWS_DIR= ${env.SYNC_AWS_DIR}"
                        if(fileExists(env.SYNC_AWS_DIR) == false) {
                            sh "mkdir ${env.SYNC_AWS_DIR}"
                        }
                            
                        if(fileExists(env.AWS_CONFIG_DIR) == false) {
                            sh "mkdir ${env.AWS_CONFIG_DIR}"
                        }

                        echo env.AWS_CONFIG_DIR
                        echo "Syncing data ..."
                        long start_time = System.currentTimeMillis()/1000
                        //sh "docker run --rm -v ${AWS_CONFIG_DIR}:/root/.aws -v ${SYNC_WORKSPACE}:/aws amazon/aws-cli s3 sync ${AWS_DIR} test_data"
                        env.SYNC_STATUS = sh(script: "docker run --rm --network=host -v ${AWS_CONFIG_DIR}:/root/.aws -v ${SYNC_WORKSPACE}:/aws amazon/aws-cli s3 sync ${AWS_DIR}/slam_test_data test_data/slam_test_data --delete",returnStatus: true)
                        long end_time = System.currentTimeMillis()/1000
                        long duration = end_time - start_time
                        int hour = duration/3600
                        int min = (duration - hour*3600)/60
                        int sec = duration - hour*3600 - min*60
                        env.SYNC_DURATION="${hour}h ${min}min ${sec}s"
                        echo "env.SYNC_STATUS=${env.SYNC_STATUS}"
                        
                        sh """
                        docker run --rm -v ${SYNC_AWS_DIR}:/aws --entrypoint bash amazon/aws-cli -c "find . -type d -empty -exec rmdir {} \\+ -print"
                        docker run --rm -v ${SYNC_AWS_DIR}:/aws --entrypoint bash amazon/aws-cli -c "find . -type d -empty -exec rmdir {} \\+ -print"
                        docker run --rm -v ${SYNC_AWS_DIR}:/aws --entrypoint bash amazon/aws-cli -c "find . -type d -empty -exec rmdir {} \\+ -print"
                        docker run --rm -v ${SYNC_AWS_DIR}:/aws --entrypoint bash amazon/aws-cli -c "find . -type d -empty -exec rmdir {} \\+ -print"
                        docker run --rm -v ${SYNC_AWS_DIR}:/aws --entrypoint bash amazon/aws-cli -c "find . -type d -empty -exec rmdir {} \\+ -print"
                        docker run --rm -v ${SYNC_AWS_DIR}:/aws --entrypoint bash amazon/aws-cli -c "find . -type d -empty -exec rmdir {} \\+ -print" """

                        if (env.SYNC_STATUS == '0') {
                            echo "Data synchronization is successful "
                            env.SYNC_RESULT='SUCCESS'
                            // Create a folder to store the data of this build
                            env.CURRENT_RESULT_DIR="${env.RESULT_DIR}/test_result_${_version}"
                            sh "mkdir ${env.CURRENT_RESULT_DIR}"
                            //build job: 'bot_data_test_results', parameters: [string(name: 'TODAY_RESULT_DIR', value: "salm_test_results_${_version}")]
                            echo "The folder path created this time:$env.CURRENT_RESULT_DIR"
                        }
                        else {
                            echo "Data synchronization failed"
                            env.SYNC_RESULT='FAILURE'
                            currentBuild.result='FAILURE'
                        }

                    }

                }
                
            }
            
        }

        stage('run slam_data_test') {
            environment {
                SLAM_DATASET = "${WORKSPACE}/../sync_bot_slam_data/test_data/slam_test_data"
            }
            when {
                expression {
                    if (env.slam_data_test == "false") {
                        echo "close slam_data_test"
                        env.SLAM_DURATION='0'
                        env.SLAM_RESULT='CLOSED'
                    }
                    return (env.SYNC_STATUS == '0' && env.slam_data_test == "true")
                }
            }
            steps {
                dir('./base') {
                    script {
                        sh "git clean -xdf"
                        try {
                            def TODAY_RESULT_DIR="${env.CURRENT_RESULT_DIR}"
                            echo "Folder to receive results：$TODAY_RESULT_DIR"
                            // sh "mkdir $TODAY_RESULT_DIR"
                            long start_time = System.currentTimeMillis()/1000
                            echo "--------------------------------"
                            echo "SLAM_DATASET： ${SLAM_DATASET}"
                            env.SLAM_STATUS = sh(script: "./tools/test/slam_test/slam_test.sh ${SLAM_DATASET} ${TODAY_RESULT_DIR}",returnStatus: true)   
                            "cp ${TODAY_RESULT_DIR}/* ${env.RESULT_DIR}/test_result/"                         
                            long end_time = System.currentTimeMillis()/1000
                            long duration = end_time - start_time
                            int hour = duration/3600
                            int min = (duration - hour*3600)/60
                            int sec = duration - hour*3600 - min*60
                            env.SLAM_DURATION="${hour}h ${min}min ${sec}s"    
                            if (env.SLAM_STATUS == '0') {
                                echo "run slam_data_test success"
                                env.SLAM_RESULT='SUCCESS'
                            }
                            else {
                                echo "run slam_data_test failure"
                                env.SLAM_RESULT='FAILURE'
                                currentBuild.result='FAILURE'
                            }                         
                        }
                        catch(ex) {
                            echo "run slam_data_test failure"
                            env.SLAM_RESULT='FAILURE'
                        }
                    }
                }
            }
        }
    


   
        stage('post email') {
            steps {
                script {
    
                    def backend_dict_response = [
                        slam_result: "Not tested",
     
                        slam_duration: "0",
    
                        build_url: env.BUILD_URL,
                        build_result_url: env.build_result_url,
                        
                        email_list: env.email_list,
                    ]
    
                    if (env.SYNC_STATUS == '0') {
                        backend_dict_response['data_synchronization_result'] = env.SYNC_RESULT
                        backend_dict_response['slam_result'] = env.SLAM_RESULT
    
                        backend_dict_response['data_synchronization_duration'] = env.SYNC_DURATION
                        backend_dict_response['slam_duration'] = env.SLAM_DURATION
                    
                    }
    
                    // post pr list
                    // def post_request = httpRequest url: env.host, contentType: 'APPLICATION_JSON', validResponseCodes: '200, 400', httpMode: 'POST', requestBody: JsonOutput.toJson(backend_dict_response)
                    // println(post_request.content)
                }
            }
        }
    }
    

    post {
        always {
            script {
                if (env.SYNC_STATUS != '0') {
                    env.SLAM_RESULT = "Not tested"
                    env.SLAM_DURATION = "0"
                }
            }

            emailext (
                
                subject: "Trifo SLAM Test Report",
                mimeType:'text/html',
                body: """
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>Trifo Test Daily Report</title>
</head>

<body leftmargin="8" marginwidth="0" topmargin="8" marginheight="4"
offset="0">
<table width="95%" cellpadding="0" cellspacing="0"
style="font-size: 11pt; font-family: Tahoma, Arial, Helvetica, sans-serif">
<tr>
<td>(This email is sent automatically by the program)</td>
</tr>
<tr>
<td><h3>
<font color="#0000FF">Data synchronization result :</font> <font color="#FF0000">${env.SYNC_RESULT}</font>      <font color="#0000FF">duration :</font> <font color="#FF0000">${env.SYNC_DURATION}</font><br/>
<font color="#0000FF">slam_data_test build result :</font> <font color="#FF0000">${env.SLAM_RESULT}      <font color="#0000FF">duration :</font> <font color="#FF0000">${env.SLAM_DURATION}</font><br/>
</h3></td>
</tr>
<tr>
<td><br />
<b><font color="#0B610B">Build result data link</font></b>
<hr size="2" width="100%" align="center" /></td>
</tr>
<tr>
<td>
<ul>
<li>Result link ： <a href="https://trifo-kubernetes.ml/jenkins/job/bot_slam_test_results/ws/test_result/slam_test_results/">https://trifo-kubernetes.ml/jenkins/job/bot_slam_test_results/ws/test_result/slam_test_results</a></li>
</ul>
</td>
</tr>
<tr>
<td><br />
<b><font color="#0B610B">Build information</font></b>
<hr size="2" width="100%" align="center" /></td>
</tr>
<tr>
<td>
<ul>
<li>project name ： ${env.JOB_NAME}</li>
<li>Build number ： ${BUILD_NUMBER}th build</li>
<li>Build branch ： ${dev_tag}</li>
<li>Build log： <a href="${BUILD_URL}console">${BUILD_URL}console</a></li>
<li>Build Url ： <a href="${BUILD_URL}">${BUILD_URL}</a></li>
<li>History change record : <a href="${RUN_CHANGES_DISPLAY_URL}">${RUN_CHANGES_DISPLAY_URL}</a></li>
</ul>
</td>
</tr>
<tr>
<td>
<ul>
</table>
</body>
</html>
                """,
                to: "${env.email_list}",
                from: "Trifo Test <test@trifo.com>"
            )
        }
        
    }
}
