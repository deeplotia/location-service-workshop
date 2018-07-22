#!groovy
import java.time.ZoneOffset
import java.time.ZonedDateTime

node('') {
    def gitCommitId = null          // filled in Compile & test stage
    def buildTimestamp = null
    def dockerImageId = "location-workshop/location-service"
    def mavenImage = docker.image('maven:3.5.2-jdk-8')
    def postgresImage = docker.image('postgres:9.6')
    def locationServiceImage = null

    stage('compile & unit-test') {
        gitCommitId = checkout(scm).GIT_COMMIT.substring(0, 7)
        buildTimestamp = ZonedDateTime.now(ZoneOffset.UTC).toString()

        dir('location-service') {
            mavenImage.inside("--entrypoint=''"){ // disable the entrypoint in the maven container, we run our own command
                sh "mvn clean verify"
            }
            junit '**/target/surefire-reports/*.xml'
        }
    }

    stage('build docker image') {
        locationServiceImage = docker.build "${dockerImageId}:${gitCommitId}", "--build-arg git_commit_id=${gitCommitId} --build-arg build_timestamp=${buildTimestamp} -f location-service/Dockerfile location-service"
    }

    stage('system tests') {
        dir('location-service-system-test') {
            postgresImage.withRun("-e POSTGRES_DATABASE=locationservice " +
                    " -e POSTGRES_USER=locationservice " +
                    " -e POSTGRES_PASSWORD=locationservice") { db ->

                locationServiceImage.withRun("--link ${db.id}:db" +
                        " -e DB_URL=jdbc:postgresql://db/locationservice") { service ->
                    // --link adds an entry into the /etc/hosts file and maps *db* to the IP address of the postgres container

                    maven.inside("--link ${service.id}:service" +
                            " --entrypoint='' " +
                            " -e LOCATION_SERVICE_ADDRESS=service:8080") {
                        // --link adds an entry into the /etc/hosts file and maps *service* to the IP address of the postgres container
                        sh "mvn clean verify"
                    }
                }
            }
            junit '**/target/surefire-reports/*.xml'
        }
    }
}