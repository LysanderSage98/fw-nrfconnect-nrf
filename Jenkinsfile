// Due to JENKINS-42369 we put these defines outside the pipeline
def IMAGE_TAG = "ncs-toolchain:1.09"
def REPO_CI_TOOLS = "https://github.com/zephyrproject-rtos/ci-tools.git"
def REPO_CI_TOOLS_SHA = "9f4dc0be401c2b1e9b1c647513fb996bd8abd057"

// Function to get the current repo URL, to be propagated to the downstream job
def getRepoURL() {
  dir('nrf') {
    sh "git config --get remote.origin.url > .git/remote-url"
    return readFile(".git/remote-url").trim()
  }
}

def check_and_store_sample(path, new_name) {
  script {
    if (fileExists(file_path)) {
      sh "cp ${path} artifacts/${new_name}"
    }
    else {
      echo "Build for ${new_name} failed"
      currentBuild.result = 'FAILURE'
    }
  }
}

pipeline {
  agent {
    docker {
      image "$IMAGE_TAG"
      label "docker && build-node && ncs && linux"
      args '-e PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/workdir/.local/bin'
    }
  }
  options {
    // Checkout the repository to this folder instead of root
    checkoutToSubdirectory('nrf')
  }

  environment {
      // ENVs for check-compliance
      GH_TOKEN = credentials('nordicbuilder-compliance-token') // This token is used to by check_compliance to comment on PRs and use checks
      GH_USERNAME = "NordicBuilder"
      COMPLIANCE_ARGS = "-r NordicPlayground/fw-nrfconnect-nrf"
      COMPLIANCE_REPORT_ARGS = "-p $CHANGE_ID -S $GIT_COMMIT -g"

      // Build all custom samples that match the ci_build tag
      SANITYCHECK_OPTIONS = "--board-root $WORKSPACE/nrf/boards --testcase-root $WORKSPACE/nrf/samples --testcase-root $WORKSPACE/nrf/applications --build-only --disable-unrecognized-section-test -t ci_build --inline-logs"
      ARCH = "-a arm"
      LC_ALL = "C.UTF-8"

      // ENVs for building (triggered by sanitycheck)
      ZEPHYR_TOOLCHAIN_VARIANT = 'gnuarmemb'
      GNUARMEMB_TOOLCHAIN_PATH = '/workdir/gcc-arm-none-eabi-7-2018-q2-update'

      // Projects to trigger after this one is built
      DOWNSTREAM_PROJECTS = credentials('fw-nrfconnect-nrf-jobs')
  }

  stages {
    stage('Checkout repositories') {
      steps {
        // Fetch the tools used to checking compliance
        dir("ci-tools") {
          git branch: "master", url: "$REPO_CI_TOOLS"
          sh "git checkout ${REPO_CI_TOOLS_SHA}"
        }
        // Initialize west
        sh "west init -l nrf/"

        // Checkout
        sh "west update"
      }
    }

    stage('Testing') {
      parallel {
        stage('Build samples') {
          steps {
            // Create a folder to store artifacts in
            sh 'mkdir artifacts'

            // Build all the samples
            dir('zephyr') {
              sh "source zephyr-env.sh && ./scripts/sanitycheck $SANITYCHECK_OPTIONS"
            }

            script {
              /* Rename the nrf52 desktop samples */
              desktop_platforms = ['nrf52840_pca20041', 'nrf52_pca20037', 'nrf52840_pca10059']
              for(int i=0; i<desktop_platforms.size(); i++) {
                file_path = "zephyr/sanity-out/${desktop_platforms[i]}/nrf_desktop/test/zephyr/zephyr.hex"
                check_and_store_sample("$file_path", "nrf_desktop_${desktop_platforms[i]}.hex")
                file_path = "zephyr/sanity-out/${desktop_platforms[i]}/nrf_desktop/test_zrelease/zephyr/zephyr.hex"
                check_and_store_sample("$file_path", "nrf_desktop_${desktop_platforms[i]}_ZRelease.hex")
              }

              /* Rename the nrf9160 samples */
              samples = ['spm']
              for(int i=0; i<samples.size(); i++)
              {
                file_path = "zephyr/sanity-out/nrf9160_pca10090/nrf9160/${samples[i]}/test_build/zephyr/zephyr.hex"
                check_and_store_sample("$file_path", "${samples[i]}_nrf9160_pca10090.hex")
              }
              ns_samples = ['lte_ble_gateway', 'at_client']
              for(int i=0; i<ns_samples.size(); i++)
              {
                file_path = "zephyr/sanity-out/nrf9160_pca10090ns/nrf9160/${ns_samples[i]}/test_build/zephyr/zephyr.hex"
                check_and_store_sample("$file_path", "${ns_samples[i]}_nrf9160_pca10090ns.hex")
              }
              ns_apps = ['asset_tracker']
              for(int i=0; i<ns_apps.size(); i++)
              {
                file_path = "zephyr/sanity-out/nrf9160_pca10090ns/${ns_apps[i]}/test_build/zephyr/zephyr.hex"
                check_and_store_sample("$file_path", "${ns_apps[i]}_nrf9160_pca10090ns.hex")
              }

            }
            archiveArtifacts allowEmptyArchive: true, artifacts: 'artifacts/*.hex'
          }
        }

        stage('Run compliance check') {
          steps {
            // Define a Groovy script block, which allows things like try/catch and if/else. If not, the junit command will not be run if check-compliance fails
            dir('nrf') {
              script {
                // If we're a pull request, compare the target branch against the current HEAD (the PR), and also report issues to the PR
                if (env.CHANGE_TARGET) {
                  COMMIT_RANGE = "origin/${env.CHANGE_TARGET}..HEAD"
                  COMPLIANCE_ARGS = "$COMPLIANCE_ARGS $COMPLIANCE_REPORT_ARGS"
                }
                // If not a PR, it's a non-PR-branch or master build. Compare against the origin.
                else {
                  COMMIT_RANGE = "origin/${env.BRANCH_NAME}..HEAD"
                }
                // Run the compliance check
                try {
                  sh "(source ../zephyr/zephyr-env.sh && ../ci-tools/scripts/check_compliance.py $COMPLIANCE_ARGS --commits $COMMIT_RANGE)"
                }
                finally {
                  junit 'compliance.xml'
                  archiveArtifacts artifacts: 'compliance.xml'
                }
              }
            }
          }
        }
      }
    }

    stage('Trigger testing build') {
      steps {
        script {
          if (env.CHANGE_TITLE) {
            PR_NAME = "${env.CHANGE_TITLE}"
          }
          else {
            PR_NAME = "$BRANCH_NAME"
          }
          def projs = [:]
          env.DOWNSTREAM_PROJECTS.split(',').each {
            projs["${it}"] = {
              build job: "${it}", propagate: true, wait: false, parameters: [string(name: 'branchname', value: "$BRANCH_NAME"),
                                                                             string(name: 'API_URL', value: "${getRepoURL()}"),
                                                                             string(name: 'API_COMMIT', value: "$GIT_COMMIT"),
                                                                             string(name: 'API_PR_NAME', value: "$PR_NAME"),
                                                                             string(name: 'API_RUN_NUMBER', value: "$BUILD_NUMBER")]
            }
          }
          parallel projs
        }
      }
    }
  }

  post {
    always {
      // Clean up the working space at the end (including tracked files)
      cleanWs()
    }
  }
}
