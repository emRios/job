pipeline {
    agent any
    environment {
        IMAGEN = 'v3-api-admision'
        PUERTO = '5325'
		CONTENEDOR_QA = 'migracion'
        CONTENEDOR_PROD = 'sermesa-services'
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'QA' || env.BRANCH_NAME == 'master') {
                        
                    }
                }
            }
        }

        stage ('Eliminación de contenedores') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'QA') {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh "docker rm -f ${IMAGEN.toLowerCase()}"
                        }
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh "docker rmi ${IMAGEN.toLowerCase()}"
                        }
                    } else {
                        echo 'Sin eliminación de contenedor'
                    }
                }
            }
        }

        stage('Lanzamiento QA DK') {
            when {
                branch 'QA'
            }
            steps {
				dir("${IMAGEN}") {
					withCredentials([string(credentialsId: 'proxy', variable: 'PROXY')]) {
						script {
							Exception caughtException = null
							catchError(buildResult: 'SUCCESS', stageResult: 'ABORTED') {
								try {
									sh "docker build  --build-arg ARG_IMAGEN=${IMAGEN} --build-arg HTTP_PROXY=$PROXY --build-arg HTTPS_PROXY=$PROXY -t ${IMAGEN.toLowerCase()} ."
									sh "docker run --restart=always -p ${PUERTO}:8080 --name ${IMAGEN.toLowerCase()} -e INSTANCIA=QA -d ${IMAGEN.toLowerCase()}"
								}catch (err) {   
									currentBuild.result = "FAILURE"
								}
							}
						}
					}
                }
            }
        }
        stage('Lanzamiento QA NC+') {
            when {
                branch 'QA'
            }
            steps {
				dir("${IMAGEN}") {
                    withCredentials([string(credentialsId: 'proxy', variable: 'PROXY'),usernamePassword(credentialsId: 'openshift_QA', passwordVariable: 'pass', usernameVariable: 'user')]) {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'ABORTED') {
                                try {
                                    sh "oc login https://api.sermesa.qa.bi.com.gt:6443 -u $user -p $pass --insecure-skip-tls-verify=true "
									sh """#!/bin/bash
                                        token=`oc whoami -t`
                                        docker login -u $user --password \$(oc whoami -t) https://default-route-openshift-image-registry.apps.sermesa.qa.bi.com.gt
                                    """
                                    sh "docker build --build-arg ARG_IMAGEN=${IMAGEN} --build-arg HTTP_PROXY=$PROXY --build-arg HTTPS_PROXY=$PROXY -t default-route-openshift-image-registry.apps.sermesa.qa.bi.com.gt/${CONTENEDOR_PROD}/${IMAGEN.toLowerCase()} ."
                                    sh "docker push default-route-openshift-image-registry.apps.sermesa.qa.bi.com.gt/${CONTENEDOR_PROD}/${IMAGEN.toLowerCase()}"
                                    sh "docker rmi -f default-route-openshift-image-registry.apps.sermesa.qa.bi.com.gt/${CONTENEDOR_PROD}/${IMAGEN.toLowerCase()}"
                                }catch (err) {   
                                    currentBuild.result = "FAILURE"
                                }
                            }
                        }
                    }
                }	
            }
        }
        stage('Lanzamiento Producción NC+') {
                    when {
                        branch 'master'
                    }
                    steps {
                        dir("${IMAGEN}") {
                            withCredentials([string(credentialsId: 'proxy', variable: 'PROXY'),usernamePassword(credentialsId: 'openshift_prodNC', passwordVariable: 'pass', usernameVariable: 'user')]) {
                                script {
                                    catchError(buildResult: 'SUCCESS', stageResult: 'ABORTED') {
                                        try {
                                            sh "oc login https://api.bancaremota.bi.com.gt:6443 -u $user -p '$pass' --insecure-skip-tls-verify=true"
                                            sh """#!/bin/bash
                                                token=`oc whoami -t`
                                                docker login -u $user -p \$token https://default-route-openshift-image-registry.apps.bancaremota.bi.com.gt/
                                            """
                                            sh "docker build --build-arg ARG_IMAGEN=${IMAGEN} --build-arg HTTP_PROXY=$PROXY --build-arg HTTPS_PROXY=$PROXY -t default-route-openshift-image-registry.apps.bancaremota.bi.com.gt/${CONTENEDOR_PROD}/${IMAGEN.toLowerCase()} ."
                                            sh "docker push default-route-openshift-image-registry.apps.bancaremota.bi.com.gt/${CONTENEDOR_PROD}/${IMAGEN.toLowerCase()}"
                                            sh "docker rmi -f default-route-openshift-image-registry.apps.bancaremota.bi.com.gt/${CONTENEDOR_PROD}/${IMAGEN.toLowerCase()}"

                                        }catch (err) {   
                                            currentBuild.result = "FAILURE"
                                        }
                                    }
                                }
                            }
                        }
                    }
                }         


    }
}

def notifyBuild(Integer estadoNotificacion = 1, String mensaje = '') {
    /*
    1: Inicial
    2: Finalización OK
    3: Finalización Error
    */
    def String canal = '#migración'
    def String gitCommitMsg
    def String gitCommitName
    def String details
    def String icono

    if (estadoNotificacion == 1) {
        gitCommitMsg =  sh (script: 'git log -1 --pretty=%B ${GIT_COMMIT}', returnStdout: true).trim()
        gitCommitName =  sh (script: 'git --no-pager show -s --format=\'%ae\'', returnStdout: true).trim()
        details =
        """${env.JOB_NAME} [${env.BUILD_NUMBER}]
        Creado: ${gitCommitName}
        Commit: '${gitCommitMsg}'"""
        color = 'BLUE'
        colorCode = '#3498db'
    }else {
        canal = env.slackId
        if (estadoNotificacion == 2) {
            icono = '🚀'
            color = 'GREEN'
            colorCode = '#00FF00'
            details = 'Finalización Exitosa'
        } else {
            icono = '🐛'
            color = 'RED'
            colorCode = '#FF0000'
            details ="‼ ${mensaje}"
        }
    }

    if (env.slackChannel) {
        details = """
                (${icono}) ${env.slackMensaje}
                ${details}
            """
        slackSend(channel: env.slackChannel, color: colorCode, message: details, timestamp: env.slackTs)
    } else {
        def slack = slackSend (channel: canal, color: colorCode, message: details)
        env.slackMensaje = details
        env.slackId = slack.threadId
        env.slackChannel = slack.channelId
        env.slackTs = slack.ts
    }
}