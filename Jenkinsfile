properties([[$class: 'ParametersDefinitionProperty', parameterDefinitions: [
[$class: 'BooleanParameterDefinition', name: 'skipTests', defaultValue: false],
[$class: 'BooleanParameterDefinition', name: 'skipDocker', defaultValue: false]
]]])

stage 'Test'
if (Boolean.valueOf(skipTests)) {
	echo "Skipped"
} else {
	def splits = splitTests parallelism: [$class: 'CountDrivenParallelism', size: 10], generateInclusions: true
	def branches = [:]
	for (int i = 0; i < splits.size(); i++) {
	  def split = splits[i]
	  branches["split${i}"] = {
	    node('dockerSlave') {
	      checkout scm
	      writeFile file: (split.includes ? 'inclusions.txt' : 'exclusions.txt'), text: split.list.join("\n")
	      writeFile file: (split.includes ? 'exclusions.txt' : 'inclusions.txt'), text: ''
	      def mvnHome = tool 'M3'
	      sh "${mvnHome}/bin/mvn -B clean test -Dmaven.test.failure.ignore"
	      step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/*.xml'])
	    }
	  }
	}
	parallel branches
}

node('dockerSlave') {
    def mvnHome = tool 'M3'
    
    sh "rm -rf *"
    sh "rm -rf .git"
    checkout scm
    checkout([$class: 'GitSCM', branches: [[name: '*/' + env.BRANCH_NAME]],
        extensions: [[$class: 'CleanCheckout'],[$class: 'LocalBranch', localBranch: env.BRANCH_NAME]]])

    stage 'Set Version'
    def originalV = version();
    def major = originalV[1];
    def minor = originalV[2];
    def patch  = Integer.parseInt(originalV[3]) + 1;
    def v = "${major}.${minor}.${patch}"
    if (v) {
       echo "Building version ${v}"
    }
    sh "${mvnHome}/bin/mvn -B versions:set -DgenerateBackupPoms=false -DnewVersion=${v}"
    sh 'git add .'
    sh "git commit -m 'Raise version'"
    sh "git tag v${v}"

    stage 'Release Build'
    sshagent(['vgulch-github']) {
      sh "${mvnHome}/bin/mvn -B -DskipTests clean deploy"
      sh "git push origin " + env.BRANCH_NAME
      sh "git push origin v${v}"
    }

    stage 'Docker Build'
    if (Boolean.valueOf(skipDocker)) {
      echo "Skipped"
    } else {
	    withEnv(['DOCKER_HOST=tcp://lab1:5000']) {
	        sh "captain build"
	        sh "captain push"
	    }
    }
}

def version() {
    def matcher = readFile('pom.xml') =~ '<version>(\\d*)\\.(\\d*)\\.(\\d*)(-SNAPSHOT)*</version>'
    matcher ? matcher[0] : null
}
