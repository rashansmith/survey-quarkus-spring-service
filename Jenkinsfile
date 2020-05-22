library identifier: "pipeline-library@v1.5",
retriever: modernSCM(
  [
    $class: "GitSCMSource",
    remote: "https://github.com/redhat-cop/pipeline-library.git"
  ]
)

openshift.withCluster() {
  env.NAMESPACE = openshift.project()
  env.APP_NAME = "survey-service"
  echo "Starting Pipeline for ${APP_NAME}..."
  env.DEV = env.NAMESPACE.replaceAll(/-ci-cd/, '-dev')
  env.TEST = env.NAMESPACE.replaceAll(/-ci-cd/, '-test')
  env.DEMO = env.NAMESPACE.replaceAll(/-ci-cd/, '-demo')
}

pipeline {
  // Use Jenkins Maven slave
  // Jenkins will dynamically provision this as OpenShift Pod
  // All the TESTs and steps of this Pipeline will be executed on this Pod
  // After Pipeline completes the Pod is killed so every run will have clean
  // workspace
  agent {
    label 'jenkins-slave-mvn'
  }

  // Pipeline TESTs start here
  // Requeres at least one TEST
  stages {

    // Run Maven build, skipping tests
    stage('Build'){
      steps {
        sh "mvn -B clean install -DskipTests=true"
      }
    }

    // Run Maven unit tests
    stage('Unit Test'){
      steps {
        sh "mvn -B test"
      }
    }

    // Build Container Image using the artifacts DEMOuced in previous TESTs
    stage('Build Container Image'){
      steps {
        // Copy the resulting artifacts into common directory
        sh """
          ls target/*
          rm -rf oc-build && mkdir -p oc-build/deployments
          for t in \$(echo "jar;war;ear" | tr ";" "\\n"); do
            cp -rfv ./target/*.\$t oc-build/deployments/ 2> /dev/null || echo "No \$t files"
          done
        """

        // Build container image using local Openshift cluster
        // Giving all the artifacts to OpenShift Binary Build
        // This places your artifacts into right location inside your S2I image
        // if the S2I image supports it.
        binaryBuild(projectName: env.NAMESPACE, buildConfigName: env.APP_NAME, buildFromPath: "oc-build")
      }
    }

    stage('Promote from Build to Dev') {
      steps {
        tagImage(sourceImageName: env.APP_NAME, sourceImagePath: env.NAMESPACE, toImagePath: env.DEV, toImageName: 'spring-service')
      }
    }

    stage('Verify Deployment to Dev') {
      steps {
        verifyDeployment(projectName: env.DEV, targetApp: 'spring-service')
      }
    }
  }
}