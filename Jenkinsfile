import java.text.SimpleDateFormat

node {
currentBuild.result = "SUCCESS"
  try{
   def mvnHome;
   def project_id;
   def artifact_id;
   def aws_s3_bucket_name;
   def aws_s3_bucket_region;
   def timeStamp;
   def userInput = false;
   def didTimeout = false;
   def baseDir;
   def currentBranch;
   def deploy_hosts;
   
   stage('Initalize'){
	   aws_s3_bucket_name = 'jvcdp-repo';
	   aws_s3_bucket_region = 'ap-southeast-1';
	   timeStamp = getTimeStamp();
	   baseDir = pwd();
	   currentBranch = env.BRANCH_NAME;
	   deploy_env=getTargetEnv(currentBranch);
       deploy_hosts = 'all'; //right now our infra is simple single ec2 for each machine so our inventory has only one host. So running it on all.

   }
   stage('Checkout') { // for display purposes
      checkout scm;
   }
   stage('Install Binaries'){
       run_playbook('install_binaries.yml', deploy_env, deploy_hosts);
   }
   stage('Install Apache'){
       run_playbook('installapache.yaml', deploy_env,deploy_hosts);
   }
   stage('Configure Apache'){
       run_playbook('configureapache.yaml', deploy_env,deploy_hosts);
   }
   stage('Install Tomcat'){
       run_playbook('installtomcat.yaml', deploy_env,deploy_hosts);
   }
   stage('Configure Tomcat'){
       run_playbook('configuretomcat.yaml', deploy_env,deploy_hosts);
   }
   stage('Test DB Connection'){
       run_playbook('testdbconnection.yaml', deploy_env,deploy_hosts);
   }
   stage('Setup DB'){
       run_playbook('setuppostgresdb.yaml', deploy_env,deploy_hosts);
   }
   stage('Deploy App. War'){
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
    accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
    credentialsId: 's3repoadmin', 
    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']])  
     {
         //Download the API(s)
            sh "echo 'ansible-playbook  -i ${deploy_env} s3downloadapis.yaml --extra-vars deploy_host=${deploy_hosts}'";
            sh "ansible-playbook  -i ${deploy_env} s3downloadapis.yaml --extra-vars 'deploy_host=${deploy_hosts}'" ;
         //Deploy the API(s)
            sh "echo 'ansible-playbook  -i ${deploy_env} deployapis.yaml --extra-vars deploy_host=${deploy_hosts}'";
            sh "ansible-playbook  -i ${deploy_env} deployapis.yaml --extra-vars 'deploy_host=${deploy_hosts}'" ;
     }
   }
  }catch (err) {

        currentBuild.result = "FAILURE"

        throw err
    }
}

def checkAndRunPlaybook(String playbook_name, String deploy_env, String deploy_hosts){
	userInput = false;
	didTimeout = false;
	
	try {
	    timeout(time: 2, unit: 'DAYS') { // change to a convenient timeout for you
	        userInput = input(message: "Should I run Playbook: ${playbook_name} to on Environment: ${deploy_env} now?", ok: 'Proceed', 
                        parameters: [booleanParam(defaultValue: true, 
                        description: 'If you would want to run the playbook now, just tick the checkbox and click Proceed!',name: 'Yes?')])
	    }
	} catch(err) { // timeout reached or input false
	    def user = err.getCauses()[0].getUser()
	    if('SYSTEM' == user.toString()) { // SYSTEM means timeout.
	        didTimeout = true
	    } else {
	        userInput = false
	        echo "Aborted by: [${user}]"
	    }
	}
        if ((didTimeout)||(!userInput)) {
            // do something on timeout
            echo "no input was received before timeout"
            currentBuild.result = 'ABORTED'
        } else {
       run_playbook(playbook_name, deploy_env, deploy_hosts);
        }
}

def getTimeStamp(){
	def dateFormat = new SimpleDateFormat("yyyyMMddHHmm")
	def date = new Date()
	return dateFormat.format(date);
}

def version() {
    def ver = readFile('pom.xml') =~ '<version>(.+)</version>'
    ver ? ver[0][1] : null
    def art = readFile('pom.xml') =~ '<artifactId>(.+)</artifactId>'
    art ? art[0][1] : null
    def pck = readFile('pom.xml') =~ '<packaging>(.+)</packaging>'
    pck ? pck[0][1] : null
	version = art+ver ? art + '-' + ver + '.' + pck : artifactId + '.jar'
	return version;
}

def runTests(duration) {
    node {
        checkout scm
        runWithServer {url ->
            mvn "-o -f sometests test -Durl=${url} -Dduration=${duration}"
        }
    }
}


def run_playbook(String playbook_name, String deploy_env, String deploy_hosts) {
    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
    accessKeyVariable: 'AWS_ACCESS_KEY_ID', 
    credentialsId: 'ec2-deploy-admin', 
    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']])  
     {
        sh "echo 'ansible-playbook  -i ${deploy_env} ${playbook_name} --extra-vars deploy_host=${deploy_hosts}'";
        sh "ansible-playbook  -i ${deploy_env} ${playbook_name} --extra-vars 'deploy_host=${deploy_hosts}'" ;
     }
}

def getTargetEnv(String branchName){
	def deploy_env="dev";
	switch(branchName){
		case('develop'):
		deploy_env="dev";
		break;
		case('sit'):
		deploy_env="sit";
		break;
		case('uat'):
		deploy_env="uat";
		break;
		case('release'):
		deploy_env="release";
		break;
		case('master'):
		deploy_env="prod";
		break;
		case('cdp'):
		deploy_env="all";
		break;
		default:
			if(branchName.startsWith("feature")){
				deploy_env="none"
			}
		break;
	}
	return deploy_env;
}

def checkAndDeploy(String baseDir, String deploy_env, String deploy_hosts, String timeStamp){
	  def  c_userInput = false;
	  def c_didTimeout = false;

	if(deploy_env=="dev"){
	//Deploy directly to dev environment
		run_playbook("main.yaml",deploy_env,deploy_hosts);
	}
	else{

	try {
	    timeout(time: 5, unit: 'DAYS') { // change to a convenient timeout for you
	        c_userInput = input(message: "Do you approve deployment to ${deploy_env}?", ok: "Proceed", 
                        parameters: [booleanParam(defaultValue: true, 
                        description: "If you would want to proceed for deployment to ${deploy_env}, just tick the checkbox and click Proceed!",name: "Yes?")])
	    }
	} catch(err) { // timeout reached or input false
	    def user = err.getCauses()[0].getUser()
	    if('SYSTEM' == user.toString()) { // SYSTEM means timeout.
	        c_didTimeout = true
	    } else {
	        c_userInput = false
	        echo "Aborted by: [${user}]"
	    }
	}
        if ((c_didTimeout)||(!c_userInput)) {
            // do something on timeout
            echo "no input was received before timeout"
            currentBuild.result = 'ABORTED'
        } else {
			run_playbook("main.yaml",deploy_env,deploy_hosts);
        } 
   }
}

def stop_container(container_name) {
    sh "docker stop ${container_name}"
}

def runWithServer(body) {
    def id = UUID.randomUUID().toString()
    deploy id
    try {
        body.call "${jettyUrl}${id}/"
    } finally {
        undeploy id
    }

}
