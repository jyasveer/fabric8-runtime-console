@Library('github.com/fabric8io/fabric8-pipeline-library@master')
def utils = new io.fabric8.Utils()
def flow = new io.fabric8.Fabric8Commands()
def repo = 'fabric8-runtime-console'
def org = 'fabric8-ui'
def ciDeploy = false
def tempVersion
def imageName
fabric8UITemplate{
    dockerNode{
        ws {
            if (utils.isCI()){
                checkout scm
                readTrusted 'release.groovy'
                def pipeline = load 'release.groovy'

                container('ui'){
                    pipeline.ci()
                }

                // container('ui'){
                //     tempVersion = pipeline.ciBuildDownstreamProject()
                // }

                // imageName = "fabric8/fabric8-ui:${tempVersion}"
                // container('docker'){
                //     pipeline.buildImage(imageName)
                // }

                //ciDeploy = true

            } else if (utils.isCD()){

                git "https://github.com/${org}/${repo}.git"
                readTrusted 'release.groovy'
                sh "git remote set-url origin git@github.com:${org}/${repo}.git"
                def pipeline = load 'release.groovy'

                container('ui'){
                    pipeline.ci()
                }

                def branch
                container('ui'){
                    branch = utils.getBranch()
                }
                
                def published
                container('ui'){
                    published = pipeline.cd(branch)
                }

                def releaseVersion
                container('ui'){
                    releaseVersion = utils.getLatestVersionFromTag()
                }

                if (published){
                    pipeline.updateDownstreamProjects(releaseVersion)
                }
            } 
        }
    }
}

// deploy a snapshot fabric8-ui pod and notify pull request of details
if (ciDeploy){
   def prj = 'fabric8-ui-'+ env.BRANCH_NAME
   prj = prj.toLowerCase()
   def route
   deployOpenShiftNode(openshiftConfigSecretName: 'fabric8-intcluster-config'){
       stage("deploy ${prj}"){
           route = deployOpenShiftSnapshot{
               mavenRepo = 'http://central.maven.org/maven2/io/fabric8/online/apps/fabric8-ui'
               githubRepo = 'fabric8-ui'
               originalImageName = 'registry.devshift.net/fabric8io/fabric8-ui'
               newImageName = imageName
               openShiftProject = prj
           }
       }
       stage('notify'){
           def changeAuthor = env.CHANGE_AUTHOR
           if (!changeAuthor){
               error "no commit author found so cannot comment on PR"
           }
           def pr = env.CHANGE_ID
           if (!pr){
               error "no pull request number found so cannot comment on PR"
           }
           def message = "@${changeAuthor} snapshot fabric8-ui is deployed and available for testing at http://${route}"
           container('clients'){
               flow.addCommentToPullRequest(message, pr, 'fabric8-ui/fabric8-runtime-console')
           }
       }
   }
}