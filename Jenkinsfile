 def azureUtil = load './deployment/jenkins/azureutil.groovy'
 azureUtil.chris()

/* Setup Variables */

// Deploy
def deploy = env.deploy.toBoolean()

// Github
def github_repository = env.github_repository
def github_branch = env.github_branch
def github_version = env.github_version

// Sonarqube
def sq_project = env.sq_project
project_key = "org.teradata.$sq_project"

// AWS
def environment = env.environment
def build_number = env.BUILD_NUMBER
def service_name = env.service_name
s3_bucket = "td-artifacts-le"
release_folder = "release-${github_version}.${build_number}"
complete_bucket_path = "${s3_bucket}/${service_name}-service/${environment}/${release_folder}"
lambda_bucket_path = "${complete_bucket_path}/lambda"
cft_bucket_path = "${complete_bucket_path}/cloudformation"
test_bucket_path = "${complete_bucket_path}/tests"
cft_s3_path = "https://s3-us-west-2.amazonaws.com/${cft_bucket_path}/${service_name}.cft.yml"

// Jenkins
green_color = "#98ec8a"
jenkins_cft_path = "infrastructure/api/${service_name}.cft.yml"

// Reusable Commands
npm_remove = "sudo rm -rf node_modules"
npm_install = "sudo npm install --unsafe-perm"
npm_build = "sudo npm run build"
npm_test = "sudo npm run test"


timestamps {
  node("New_Slave"){
    //Import Build Tools
    def tools = load "/opt/tools.groovy"

    try {

      stage("Pull Github Repository") {
        stageName = "Pull Github Repository"
        git url: "${github_repository}", branch: "${github_branch}", credentialsId: "teradata-jenkins"
      }

      stage("Install NPM Modules") {
        stageName = "Install NPM Modules"
        parallel rootDir: {
          sh "${npm_remove}"
          sh "${npm_install}"
        }, apiDir: {
          dir("api") {
            sh "${npm_remove}"
            sh "${npm_install}"
          } // dir api
        }, testDir: {
          dir("tests/api") {
            sh "${npm_remove}"
            sh "${npm_install}"
            sh "${npm_build}"
            sh "${npm_remove}" //remove node_modules to exclude from final artifact
          }
        },
        failFast: true
      }

      stage("[GATE] NPM Unit Test") {
        stageName = "[GATE] NPM Unit Test"
        dir("api") {
          sh "${npm_test}"
        }
      }
      
      stage('[GATE] Sonarqube Analysis') {
        stageName = "[GATE] Sonarqube Analysis"
        def scannerHome = tool 'SonarQube'
        withSonarQubeEnv("Sonarqube6x") {
          sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=${project_key} -Dsonar.projectName=${sq_project}"
        }
      }

      stage("Build ${service_name} Service") {
        stageName = "Build ${service_name} Service"
        dir("api") {
          timeout(5) {
            sh "${npm_build}"
          } // Timeout
        }
      }

      parallel zip: {
        stage("ZIP ${service_name} Service Lambda Functions") {
          stageName = "ZIP ${service_name} Service Lambda Functions"
          tools.zipLambda("api/dist", "${service_name}Service")
        }
      }, valCFT: {
        stage('Validate Cloud Formation Template') {
          stageName = 'Validate Cloud Formation Template'

          //Upload to tmp s3 location
          sh "aws s3 cp ${jenkins_cft_path} s3://${cft_bucket_path}/${service_name}.cft.yml"

          //Validate CFT
          sh "aws cloudformation validate-template --template-url ${cft_s3_path}"

          //Delete From tmp s3 location
          sh "aws s3 rm s3://${cft_bucket_path}/${service_name}.cft.yml"

        }
      }

      if (deploy) {
        stage("Upload Artifacts") {
          stageName = "Upload Artifacts"

          parallel lambda: {
            //lambda artifact folder upload
            step([$class: 'S3BucketPublisher', dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: "${lambda_bucket_path}", excludedFile: '*.js', flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: true, selectedRegion: 'us-west-2', showDirectlyInBrowser: false, sourceFile: 'api/dist/*.zip', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: true, userMetadata: [[key: '', value: '']]]], profileName: 'arn:aws:iam::171166726173:instance-profile/Teradata-IAM-resJenkinsS3Profile-1WA60OHB54X6D', userMetadata: []])
          }, cloudformation: {
            //cloudformation artifact folder upload
            step([$class: 'S3BucketPublisher', dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: "${cft_bucket_path}", flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: true, selectedRegion: 'us-west-2', showDirectlyInBrowser: false, sourceFile: "${jenkins_cft_path}", storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: true, userMetadata: [[key: '', value: '']]]], profileName: 'arn:aws:iam::171166726173:instance-profile/Teradata-IAM-resJenkinsS3Profile-1WA60OHB54X6D', userMetadata: []])
          }, tests: {
            //tests artifact folder upload
            step([$class: 'S3BucketPublisher', dontWaitForConcurrentBuildCompletion: false, entries: [[bucket: "${test_bucket_path}", flatten: false, gzipFiles: false, keepForever: false, managedArtifacts: false, noUploadOnFailure: true, selectedRegion: 'us-west-2', showDirectlyInBrowser: false, sourceFile: 'tests/api/', storageClass: 'STANDARD', uploadFromSlave: false, useServerSideEncryption: true, userMetadata: [[key: '', value: '']]]], profileName: 'arn:aws:iam::171166726173:instance-profile/Teradata-IAM-resJenkinsS3Profile-1WA60OHB54X6D', userMetadata: []])
          }
        }

        stage('Deploy Cloud Formation package') {
          stageName = "Deploy Cloud Formation package"

          //Check for changes in the CFT
          hasChanges = tools.CFTHasChanges(environment, service_name, release_folder, cft_s3_path)

          if (hasChanges) {
            //If there are changes or it is new deploy CFT
            tools.deployCFTChanges(environment, service_name, cft_s3_path)

        } else {
            print 'CFT does not contain changes, skipping Deploy Cloud Formation package stage!'
          }
        }

        stage ('Deploy API Gateway') {
          stageName = "Deploy API Gateway"
          tools.redeployApi(environment, service_name)
        }

        // Add success tag to inform build was deployed to Dev
        manager.addShortText("${release_folder} Deployed", "black", "${green_color}", "1px", "green")

        stage('Start Test Environment Deploy'){
          stageName = "Start Test Environment Deploy"
          build job: "${service_name}-service-test", parameters: [string(name: 'github_repository', value: "${github_repository}"), string(name: 'github_branch', value: "${github_branch}"),string(name: 'environment', value: "test"), string(name: 'release_folder', value: "${release_folder}")], wait: false
        }
      }
      else {
        manager.addShortText("${github_branch} Success", "black", "${green_color}", "1px", "green")
      } // Else Tag anyways, since it probably succeeded the feature branch.
    } // End Try Block
    catch(e) {
      currentBuild.result = 'FAILED'

      // Tag the build with the stage that failed, branch and build number
      manager.addShortText("${stageName}", "black", "#ffd833", "1px", "orange")
      manager.addShortText("${github_branch} ${release_folder}", "black", "#d2d2d2", "1px", "black")

      tools.notifyBuild(currentBuild.result)

      throw e
    } //End Catch Block
  } // End Node
} // End Timestamps
  
  
