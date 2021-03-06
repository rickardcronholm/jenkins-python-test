pipeline {
    agent any

    triggers {
        pollSCM('*/5 * * * 1-5')
    }

    options {
        skipDefaultCheckout(true)
        // Keep the 10 most recent builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    environment {
      PATH="$PATH"
    }

    stages {

        stage ("Code pull"){
            steps{
                checkout scm
            }
        }

        stage('Create environment') {
            steps {
		withPythonEnv('python3') {
                	echo "Building virtualenv"
			sh  ''' python3 -m venv venv
		          	. ./venv/bin/activate
		        	pip install -r requirements/dev.txt
			    '''
		}
            }
        }

        stage('Static code metrics') {
            steps {
		withPythonEnv('python3') {
	                echo "Raw metrics"
        	        sh  ''' . ./venv/bin/activate
                	        radon raw --json irisvmpy > raw_report.json
                        	radon cc --json irisvmpy > cc_report.json
	                        radon mi --json irisvmpy > mi_report.json
        	                sloccount --duplicates --wide irisvmpy > sloccount.sc
                	    '''
	                echo "Test coverage"
        	        sh  ''' . ./venv/bin/activate
                	        coverage run irisvmpy/iris.py 1 1 2 3
                        	python -m coverage xml -o reports/coverage.xml
	                    '''
        	        echo "Style check"
                	sh  ''' . ./venv/bin/activate
                        	pylint irisvmpy || true
	                    '''
		}
            }
            post{
                always{
                    step([$class: 'CoberturaPublisher',
                                   autoUpdateHealth: false,
                                   autoUpdateStability: false,
                                   coberturaReportFile: 'reports/coverage.xml',
                                   failNoReports: false,
                                   failUnhealthy: false,
                                   failUnstable: false,
                                   maxNumberOfBuilds: 10,
                                   onlyStable: false,
                                   sourceEncoding: 'ASCII',
                                   zoomCoverageChart: false])
                }
            }
        }



        stage('Unit tests') {
            steps {
		withPythonEnv('python3') {
	               sh  ''' . ./venv/bin/activate
        	               python -m pytest --verbose --junit-xml reports/unit_tests.xml
                	   '''
		}
            }
            post {
                always {
                    // Archive unit tests for the future
                    junit allowEmptyResults: true, testResults: 'reports/unit_tests.xml'
                }
            }
        }

        stage('Acceptance tests') {
            steps {
		withPythonEnv('python3') {
	               sh  ''' . ./venv/bin/activate
        	               behave -f=formatters.cucumber_json:PrettyCucumberJSONFormatter -o ./reports/acceptance.json || true
                	   '''
		}
            }
            post {
                always {
                    cucumber (buildStatus: 'SUCCESS',
                    fileIncludePattern: '**/*.json',
                    jsonReportDirectory: './reports/',
                    parallelTesting: true,
                    sortingMethod: 'ALPHABETICAL')
                }
            }
        }

        stage('Build package') {
            when {
                expression {
                    currentBuild.result == null || currentBuild.result == 'SUCCESS'
                }
            }
            steps {
		withPythonEnv('python3') {
	               sh  ''' . ./venv/bin/activate
        	               python setup.py bdist_wheel
                	   '''
		}
            }
            post {
                always {
                    // Archive unit tests for the future
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'dist/*whl', fingerprint: true
                }
            }
        }

        // stage("Deploy to PyPI") {
        //     steps {
        //         sh """twine upload dist/*
        //         """
        //     }
        // }
    }

    post {
        always {
            echo "I really should remove venv"
        }
        failure {
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: """<p>FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
                         <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>""",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']])
        }
    }
}
