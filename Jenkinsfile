#!/usr/bin/env groovy

userMap = [
        "tonyeiven": [
                "akId": "**",
                "akSecret": "**",
                "accountId": "**"
        ]
]

pipeline {
    agent none

    options {
        disableConcurrentBuilds()
        timeout(time: 2, unit: 'HOURS')
        timestamps()
		ansiColor('xterm')
    }
	
	parameters {
		choice(name: 'TEST_CLOUD_ENV', choices:['beta', 'staging', 'production'], description: 'hdr cloud environment')
		choice(name: 'USER_ACCOUNT', choices:userMap.keySet() as List<String>, description: 'cloud accout for testing')
	}
	
	environment {
		DINGTALK_TOKEN = credentials("dingtalk-webhook")
	}
	
    stages {

        stage('Sanity Test') {
            agent {
                docker {
                    image 'java-thrift:gradle-5.1'
                    args '-u root --privileged'
                }
            }

            environment {
                ALICLOUD_AK_ID = credentials("${getAccessKeyId()}")
                ALICLOUD_AK_SECRET = credentials("${getAccessKeySecret()}")
                ALICLOUD_ACCOUNT_ID = getAccountId()
                TEST_REGION_ID = "cn-hangzhou"
                TEST_ZONE_ID = "cn-hangzhou-g"
            }

            steps {
				sh 'env'
				sh 'gradle clean test --console=plain'
            }
        }
		stage('Result Publish') {
			agent any
			steps {
				publishHTML (target: [reportName:'CDR_TEST_RESULT', reportDir:'end2end/build/reports/tests/test/', reportFiles:'index.html', keepAll: true])
			}
		}
    }
	post {
        always {
            node('master') {
                script {
                    def result = currentBuild.currentResult == 'SUCCESS'? "成功" : "失败"

                    def text = "###### 包含变更:\n${getChangeString()}"

					def title = "Jenkins工程#${JOB_NAME}在第${BUILD_NUMBER}次的构建中${result},持续时间${currentBuild.durationString.replace(' and counting', '')}"

                    sendDingDing(title, text)
                }
            }
        }
    }
}

@NonCPS
def getAccessKeyId() {
    return userMap["${params.USER_ACCOUNT}"]["akId"]
}

@NonCPS
def getAccessKeySecret() {
    return userMap["${params.USER_ACCOUNT}"]["akSecret"]
}

@NonCPS
def getAccountId() {
    return userMap["${params.USER_ACCOUNT}"]["accountId"]
}

@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "收集变更信息中..."
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += "- *${truncated_msg} [${entry.author}]*\n"
        }
    }

    if (!changeString) {
        changeString = "  *没有新的变更*\n"
    }
    return changeString
}

def sendDingDing(title, text) {
    def dingdingMessage = readJSON file: 'dingding.json'

    dingdingMessage['markdown']['title'] = title.toString()
    dingdingMessage['markdown']['text'] = "#### ** ${title.toString()} **\n${text}\n\n[点击访问jenkins](${BUILD_URL})".toString()

    echo "about to send dingding notify:"
    echo dingdingMessage['markdown']['text']

    httpRequest url:"https://oapi.dingtalk.com/robot/send?access_token=${DINGTALK_TOKEN}", contentType:'APPLICATION_JSON_UTF8', httpMode:'POST', consoleLogResponseBody:true, requestBody:dingdingMessage.toString()
}