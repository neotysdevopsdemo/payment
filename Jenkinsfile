

pipeline {
    agent  { label 'master' }
    tools {
        go 'go'
        jdk 'jdk8'
    }
  environment {
    VERSION="0.1"
    APP_NAME = "payment"
    TAG = "neotysdevopsdemo/${APP_NAME}"
    TAG_DEV = "${TAG}:DEV-${VERSION}"
    TAG_STAGING = "${TAG}-stagging:${VERSION}"
    DYNATRACEID="${env.DT_ACCOUNTID}.live.dynatrace.com"
    DYNATRACEAPIKEY="${env.DT_API_TOKEN}"
    NLAPIKEY="${env.NL_WEB_API_KEY}"
    NL_DT_TAG="app:${env.APP_NAME},environment:dev"
    OUTPUTSANITYCHECK="$WORKSPACE/infrastructure/sanitycheck.json"
    NEOLOAD_ASCODEFILE="$WORKSPACE/test/neoload/payment_neoload.yaml"
    NEOLOAD_ANOMALIEDETECTIONFILE="$WORKSPACE/monspec/payment_anomalieDection.json"
    DOCKER_COMPOSE_TEMPLATE="$WORKSPACE/infrastructure/infrastructure/neoload/docker-compose.template"
    DOCKER_COMPOSE_LG_FILE = "$WORKSPACE/infrastructure/infrastructure/neoload/docker-compose-neoload.yml"
    BASICCHECKURI="health"
    PAYMENTURI="paymentAuth"
    GROUP = "neotysdevopsdemo"
    COMMIT = "DEV-${VERSION}"
  }
  stages {
      stage('Checkout') {
          agent { label 'master' }
          steps {
              git  url:"https://github.com/${GROUP}/${APP_NAME}.git",
                      branch :'master'
          }
      }
    stage('Go build') {
      steps {


          sh '''
            export GOPATH=$PWD

     
           mkdir -p src/github.com/neotysdevopsdemo/payment/
           cp -R ./api src/github.com/neotysdevopsdemo/payment/
           cp -R ./main.go src/github.com/neotysdevopsdemo/payment/
           cp -R ./glide.* src/github.com/neotysdevopsdemo/payment/
           cd src/github.com/neotysdevopsdemo/payment
           go get -v github.com/Masterminds/glide
          
           
         
          '''
        }

    }
    stage('Docker build') {
        steps {
            withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                sh "docker pull ${GROUP}/${APP_NAME}:DEV-0.1"
                sh "docker tag ${GROUP}/${APP_NAME}:DEV-0.1 ${TAG_DEV}"
                sh "docker login --username=${USER} --password=${TOKEN}"
                sh "docker push ${TAG_DEV}"


            }

        }

    }
     stage('create docker netwrok') {

                                      steps {
                                           sh "docker network create ${APP_NAME} || true"

                                      }
                       }

    stage('Deploy to dev namespace') {
        steps {
            sh "sed -i 's,TAG_TO_REPLACE,${TAG_DEV},'  $WORKSPACE/docker-compose.yml"
            sh "sed -i 's,TO_REPLACE,${APP_NAME},' $WORKSPACE/docker-compose.yml"
            sh 'docker-compose -f $WORKSPACE/docker-compose.yml up -d'

        }

    }

     stage('Start NeoLoad infrastructure') {

                                steps {
                                           sh "cp -f ${DOCKER_COMPOSE_TEMPLATE} ${DOCKER_COMPOSE_LG_FILE}"
                                           sh "sed -i 's,TO_REPLACE,${APP_NAME},'  ${DOCKER_COMPOSE_LG_FILE}"
                                           sh "sed -i 's,TOKEN_TOBE_REPLACE,$NLAPIKEY,'  ${DOCKER_COMPOSE_LG_FILE}"
                                           sh 'docker-compose -f ${DOCKER_COMPOSE_LG_FILE} up -d'
                                           sleep 15

                                       }

                           }

    stage('Run functional check in dev') {


      steps {
        sh "mkdir $WORKSPACE/test/neoload/load_template/custom-resources/"

        sh "cp $WORKSPACE/monspec/payment_anomalieDection.json  $WORKSPACE/test/neoload/load_template/custom-resources/"

         sh "sed -i 's/CHECK_TO_REPLACE/${BASICCHECKURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/PAYMENT_TO_REPLACE/${PAYMENTURI}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/HOST_TO_REPLACE/${env.APP_NAME}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/PORT_TO_REPLACE/80/' $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/DTID_TO_REPLACE/${DYNATRACEID}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/APIKEY_TO_REPLACE/${DYNATRACEAPIKEY}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's,JSONFILE_TO_REPLACE,payment_anomalieDection.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's/TAGS_TO_REPLACE/${NL_DT_TAG}/'  $WORKSPACE/test/neoload/payment_neoload.yaml"
         sh "sed -i 's,OUTPUTFILE_TO_REPLACE,$WORKSPACE/infrastructure/sanitycheck.json,'  $WORKSPACE/test/neoload/payment_neoload.yaml"


        sh "mkdir $WORKSPACE/test/neoload/neoload_project"
        sh "cp $WORKSPACE/test/neoload/payment_neoload.yaml $WORKSPACE/test/neoload/load_template/"
        sh "cd $WORKSPACE/test/neoload/load_template/ ; zip -r $WORKSPACE/test/neoload/neoload_project/neoloadproject.zip ./*"



        sleep 90
             sh "docker run --rm \
                                           -v $WORKSPACE/test/neoload/neoload_project/:/neoload-project \
                                           -e NEOLOADWEB_TOKEN=$NLAPIKEY \
                                           -e TEST_RESULT_NAME=FuncCheck_payment_${VERSION}_${BUILD_NUMBER} \
                                           -e SCENARIO_NAME=paymentLoad \
                                           -e CONTROLLER_ZONE_ID=defaultzone \
                                           -e AS_CODE_FILES=payment_neoload.yaml \
                                           -e LG_ZONE_IDS=defaultzone:1 \
                                           --network ${APP_NAME} --user root \
                                            neotys/neoload-web-test-launcher:latest"
          /*script {
              neoloadRun executable: '/home/neoload/neoload/bin/NeoLoadCmd',
                      project: "$WORKSPACE/test/neoload/load_template/load_template.nlp",
                      testName: 'FuncCheck_payment_${VERSION}_${BUILD_NUMBER}',
                      testDescription: 'FuncCheck_payment_${VERSION}_${BUILD_NUMBER}',
                      commandLineOption: "-project  $WORKSPACE/test/neoload/payment_neoload.yaml -nlweb -L paymentLoad=$WORKSPACE/infrastructure/infrastructure/neoload/lg/remote.txt -L Population_Dynatrace_Integration=$WORKSPACE/infrastructure/infrastructure/neoload/lg/local.txt -nlwebToken $NLAPIKEY -variables host=payment,port=80",
                      scenario: 'paymentLoad', sharedLicense: [server: 'NeoLoad Demo License', duration: 2, vuCount: 200],
                      trendGraphs: [
                              [name: 'Limit test Catalogue API Response time', curve: ['CatalogueList>Actions>Get Catalogue List'], statistic: 'average'],
                              'ErrorRate'
                      ]
          }*/


      }
    }
    stage('Mark artifact for staging namespace') {

      steps {

              withCredentials([usernamePassword(credentialsId: 'dockerHub', passwordVariable: 'TOKEN', usernameVariable: 'USER')]) {
                  sh "docker login --username=${USER} --password=${TOKEN}"

                  sh "docker tag ${TAG_DEV} ${TAG_STAGING}"
                  sh "docker push ${TAG_STAGING}"
              }


      }


    }

  }
   post {
       always {
           sh 'docker-compose -f $WORKSPACE/infrastructure/infrastructure/neoload/lg/docker-compose.yml down'
           sh 'docker-compose -f $WORKSPACE/docker-compose.yml down'
           cleanWs()
           sh 'docker volume prune'
       }

          }
}
