def dockerImage = 'drunkenoctopus/drunken-octopus-build-image:latest'
def configs

def parallelSteps = [:]
def md5SumsStashes=[]
def firmwareStashes=[]
node {
    stage('checkout') {
        deleteDir()
        scmResult = checkout(scm)
        stash name: 'source', includes: "**/*"
    }

    stage('config') {
        docker.image(dockerImage).inside() {
            sh './build-configs.sh'
            configs = findFiles(glob: 'config/examples/**/Configuration.h')
            for (config in configs) {
                def toolheadFile = new File(config.getPath()).parentFile
                def printerFile = toolheadFile.parentFile;
                def printer = printerFile.getName()
                def toolhead = toolheadFile.getName()

                def configurationName = "${printer}-${toolhead}"
                def md5SumsStash = "${printer}-${toolhead}-md5sum"
                md5SumsStashes.add(md5SumsStash)
                def firmwareStash = "${printer}-${toolhead}-firmware"
                firmwareStashes.add(firmwareStash)

                stash name: configurationName, includes: "config/examples/**/${printer}/${toolhead}/*.h"

                parallelSteps[configurationName] = {
                    node {
                        ws(configurationName) {
                            unstash 'source'
                            unstash configurationName
                            docker.image(dockerImage).inside() {
                                sh "./build-firmware.sh '${printer}' '${toolhead}'"
                                archiveArtifacts artifacts: "build/**/*.hex, build/**/*.bin", fingerprint: true
                                stash name: firmwareStash, includes: "build/**/*.hex, build/**/*.bin"
                            }
                        }
                    }
                }
            }
        }
    }
}


stage('build') {
    parallel(parallelSteps)

    node {
        for (firmwareStash in firmwareStashes) {
            unstash firmwareStash
        }

        firmwareFiles = findFiles(glob: "build/**/*.hex, build/**/*.bin")
        sh "md5sum -b ${firmwareFiles.join(' ')} >> build/md5sums.txt"
        archiveArtifacts artifacts: "build/md5sums.txt", fingerprint: true
    }
}