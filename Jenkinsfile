pipeline {
    agent {
        label 'master'
    }
    stages {
        stage('Clean') {
            steps {
                sh 'rm -rf out'
            }
        }
        stage('Transform') {
            agent {
                docker {
                    image 'cloudfluff/databaker'
                    reuseNode true
                }
            }
            steps {
                sh 'jupyter-nbconvert --to python --stdout Balanceofpayments2017q3_TabF.ipynb | ipython'
            }
        }
        stage('Upload draftset') {
            steps {
                script {
                    def PMD = 'https://production-drafter-ons-alpha.publishmydata.com'
                    def drafts = readJSON(text: httpRequest(acceptType: 'APPLICATION_JSON',
                                                            authentication: 'onspmd',
                                                            httpMode: 'GET',
                                                            url: "${PMD}/v1/draftsets?include=owned").content)
                    def jobDraft = drafts.find  { it['display-name'] == env.JOB_NAME }
                    if (jobDraft) {
                        def rmDraftResponse = httpRequest(acceptType: 'APPLICATION_JSON',
                                                          authentication: 'onspmd', httpMode: 'DELETE', url: "${PMD}/v1/draftset/${jobDraft.id}")
                        if (rmDraftResponse.status == 202) {
                            def rmDraftJob = readJSON(text: rmDraftResponse.content)
                            waitForJob("${PMD}${rmDraftJob['finished-job']}", rmDraftJob['restart-id'])
                        } else {
                            error "Problem deleting existing draftset ${rmDraftResponse.status} : ${rmDraftResponse.content}"
                        }
                    }
                    def newDraftResponse = httpRequest(acceptType: 'APPLICATION_JSON', authentication: 'onspmd',
                                                       httpMode: 'POST', url: "${PMD}/v1/draftsets?display-name=${env.JOB_NAME}")
                    if (newDraftResponse.status == 200) {
                        newJobDraft = readJSON(text: newDraftResponse.content)
                        String metadataGraph = "http://gss-data.org.uk/graph/ons-balance-of-payments/metadata"
                        String encGraph = java.net.URLEncoder.encode(metadataGraph, "UTF-8")
                        rmDatasetMetadataResponse = httpRequest(acceptType: 'APPLICATION_JSON',
                                                                authentication: 'onspmd', httpMode: 'DELETE',
                                                                url: "${PMD}/v1/draftset/${newJobDraft.id}/graph?graph=${encGraph}&silent=true")
                        if (rmDatasetMetadataResponse.status != 200) {
                            error "Problem deleting dataset metadata graph ${rmDatasetMetadataResponse.status} : ${rmDatasetMetadataResponse.content}"
                        }
                        addDatasetMetadataResponse = httpRequest(acceptType: 'APPLICATION_JSON',
                                                                 authentication: 'onspmd', httpMode: 'PUT',
                                                                 url: "${PMD}/v1/draftset/${newJobDraft.id}/data",
                                                                 requestBody: readFile("metadata/dataset.trig"),
                                                                 customHeaders: [[name: 'Content-Type',
                                                                                  value: 'application/trig']])
                        if (addDatasetMetadataResponse.status == 202) {
                            def addDatasetMetadataJob = readJSON(text: addDatasetMetadataResponse.content)
                            waitForJob("${PMD}${addDatasetMetadataJob['finished-job']}", addDatasetMetadataJob['restart-id'])
                        } else {
                            error "Problem adding dataset metadata ${addDatasetMetadataResponse.status} : ${addDatasetMetadataResponse.content}"
                        }
                        runPipeline(
                            'http://production-grafter-ons-alpha.publishmydata.com/v1/pipelines/ons-table2qb.core/components/import',
                            newJobDraft.id, 'onspmd',
                            [
                                [name: 'components-csv', file: [name: 'metadata/components.csv', type: 'text/csv']]
                            ]
                        )
                        runPipeline(
                            'http://production-grafter-ons-alpha.publishmydata.com/v1/pipelines/ons-table2qb.core/codelist/import',
                            newJobDraft.id, 'onspmd',
                            [
                                [name: 'codelist-csv', file: [name: 'metadata/flow-directions.csv', type: 'text/csv']],
                                [name: 'codelist-name', value: 'Flow Directions']
                            ]
                        )
                        runPipeline(
                            'http://production-grafter-ons-alpha.publishmydata.com/v1/pipelines/ons-table2qb.core/codelist/import',
                            newJobDraft.id, 'onspmd',
                            [
                                [name: 'codelist-csv', file: [name: 'metadata/services.csv', type: 'text/csv']],
                                [name: 'codelist-name', value: 'Services']
                            ]
                        )
                        runPipeline(
                            'http://production-grafter-ons-alpha.publishmydata.com/v1/pipelines/ons-table2qb.core/data-cube/import',
                            newJobDraft.id, 'onspmd',
                            [
                                [name: 'observations-csv', file: [name: 'out/balanceofpayments2017q3.csv', type: 'text/csv']],
                                [name: 'dataset-name', value: 'ONS Balance of Payments']
                            ]
                        )
                    }
                }
            }
        }
        stage('Test Draftset') {
            steps {
                echo 'Placeholder for acceptance tests from e.g. GDP-205'
            }
        }
        stage('Publish') {
            steps {
                script {
                    def PMD = 'https://production-drafter-ons-alpha.publishmydata.com'
                    def drafts = readJSON(text: httpRequest(acceptType: 'APPLICATION_JSON',
                                                            authentication: 'onspmd',
                                                            httpMode: 'GET',
                                                            url: "${PMD}/v1/draftsets?include=owned").content)
                    def jobDraft = drafts.find  { it['display-name'] == env.JOB_NAME }
                    if (jobDraft) {
                        def publishResponse = httpRequest(acceptType: 'APPLICATION_JSON',
                                                      authentication: 'onspmd',
                                                      httpMode: 'POST',
                                                      url: "${PMD}/v1/draftset/${jobDraft.id}/publish")
                        if (publishResponse.status == 202) {
                            def publishJob = readJSON(text: publishResponse.content)
                            waitForJob("${PMD}${publishJob['finished-job']}", publishJob['restart-id'])
                        } else {
                            error "Problem publishing draftset ${publishResponse.status} : ${publishResponse.content}"
                        }
                    } else {
                        error "Expecting a draftset for this job."
                    }
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts 'out/*'
        }
    }
}

void runPipeline(pipelineUrl, draftsetId, credentials, params) {
    withCredentials([usernameColonPassword(credentialsId: credentials, variable: 'USERPASS')]) {
        String boundary = UUID.randomUUID().toString()
        allParams = [
            [name: '__endpoint-type', value: 'grafter-server.destination/draftset-update'],
            [name: '__endpoint', value: groovy.json.JsonOutput.toJson([
                url: "http://localhost:3001/v1/draftset/${draftsetId}/data",
                headers: [Authorization: "Basic ${USERPASS.bytes.encodeBase64()}"]
            ])]] + params
        String body = ""
        allParams.each { param ->
            body += "--${boundary}\r\n"
            body += 'Content-Disposition: form-data; name="' + param.name + '"'
            if (param.containsKey('file')) {
                body += '; filename="' + param.file.name + '"\r\nContent-Type: "' + param.file.type + '\r\n\r\n'
                body += readFile(param.file.name) + '\r\n'
            } else {
                body += "\r\n\r\n${param.value}\r\n"
            }
        }
        body += "--${boundary}--\r\n"
        def importRequest = httpRequest(acceptType: 'APPLICATION_JSON', authentication: credentials,
                                        httpMode: 'POST', url: pipelineUrl, requestBody: body,
                                        customHeaders: [[name: 'Content-Type', value: 'multipart/form-data;boundary="' + boundary + '"']])
        if (importRequest.status == 202) {
            def importJob = readJSON(text: importRequest.content)
            String jobUrl = new java.net.URI(pipelineUrl).resolve(importJob['finished-job']) as String
            waitForJob(jobUrl, importJob['restart-id'])
        } else {
            error "Failed import, ${importRequest.status} : ${importRequest.content}"
        }
    }
}

void waitForJob(pollUrl, restartId) {
    while (true) {
        jobResponse = httpRequest(acceptType: 'APPLICATION_JSON', authentication: 'onspmd',
                                  httpMode: 'GET', url: pollUrl, validResponseCodes: '200:404')
        if (jobResponse.status == 404) {
            if (readJSON(text: jobResponse.content)['restart-id'] != restartId) {
                error "Failed waiting for job to finish, restart-id different."
            } else {
                sleep 10
            }
        } else if (jobResponse.status == 200) {
            def jobResponseObj = readJSON(text: jobResponse.content)
            if (jobResponseObj['restart-id'] != restartId) {
                error "Failed waiting for job to finish, restart-id different."
            } else if (jobResponseObj.type == 'ok') {
                return
            } else if (jobResponseObj.type == "error") {
                error "Pipeline error in ${jobResponseObj.details?.pipeline?.name}. ${jobResponseObj.message}"
            }
        }
    }
}
