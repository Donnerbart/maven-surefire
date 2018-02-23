#!/usr/bin/env groovy

/*
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *   http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing,
 * software distributed under the License is distributed on an
 * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
 * KIND, either express or implied.  See the License for the
 * specific language governing permissions and limitations
 * under the License.
 */

properties(
    [
        buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '0', daysToKeepStr: env.BRANCH_NAME == 'master' ? '30' : '7', numToKeepStr: '5')),
        disableConcurrentBuilds()
    ]
)

final def oses = ['linux', 'windows']
final def mavens = ['3.2.x', '3.3.x', '3.5.x'] // env.BRANCH_NAME == 'master' ? ['3.2.x', '3.3.x', '3.5.x'] : ['3.5.x']
final def jdks = [7, 8, 9, 10] // env.BRANCH_NAME == 'master' ? [7, 8, 9, 10] : [9, 10]

final def options = ['-e', '-V', '-nsu', '-P', 'run-its']
final def optionsIT = ['-pl', 'maven-failsafe-plugin', '-P', 'run-its']
final def goals = ['clean', 'install', 'jacoco:report']
final Map stages = [:]

oses.eachWithIndex { os, indexOfOs ->
    mavens.eachWithIndex { maven, indexOfMaven ->
        jdks.eachWithIndex { jdk, indexOfJdk ->
            final String label = jenkinsEnv.labelForOS(os);
            final String jdkTestName = jenkinsEnv.jdkFromVersion(os, jdk.toString())
            final String jdkName = jenkinsEnv.jdkFromVersion(os, '8')
            final String mvnName = jenkinsEnv.mvnFromVersion(os, maven)
            final String stageKey = "${os}-jdk${jdk}-maven${maven}"

            def mavenOpts = '-server -XX:+UseG1GC -XX:+TieredCompilation -XX:TieredStopAtLevel=1 -XX:+UseNUMA \
-Xms64m -Djava.awt.headless=true'
            if (jdk > 7) {
                mavenOpts += ' -XX:+UseStringDeduplication'
            }
            mavenOpts += (os == 'linux' ? ' -Xmx1g' : ' -Xmx256m')

            if (label == null || jdkTestName == null || mvnName == null) {
                println "Skipping ${stageKey} as unsupported by Jenkins Environment."
                return;
            }

            println "${stageKey}  ==>  Label: ${label}, JDK: ${jdkTestName}, Maven: ${mvnName}."

            stages[stageKey] = {
                node(label) {
                    ansiColor('xterm') {
                        timestamps {
                            if (os == 'linux') {
                                //https://github.com/jacoco/jacoco/issues/629
                                boolean makeReports = indexOfMaven == mavens.size() - 1 && jdk == 9
                                buildProcess(stageKey, jdkName, jdkTestName, mvnName, goals, options, optionsIT, mavenOpts, makeReports)
                            } else {
                                buildProcess(stageKey, jdkName, jdkTestName, mvnName, goals, options, optionsIT, mavenOpts, false)
                            }
                        }
                    }
                }
            }
        }
    }
}

timeout(time: 10, unit: 'HOURS') {
    try {
        parallel(stages)
        // JENKINS-34376 seems to make it hard to detect the aborted builds
    } catch (org.jenkinsci.plugins.workflow.steps.FlowInterruptedException e) {
        // this ambiguous condition means a user probably aborted
        if (e.causes.size() == 0) {
            currentBuild.result = "ABORTED"
        } else {
            currentBuild.result = "FAILURE"
        }
        throw e
    } catch (hudson.AbortException e) {
        // this ambiguous condition means during a shell step, user probably aborted
        if (e.getMessage().contains('script returned exit code 143')) {
            currentBuild.result = "ABORTED"
        } else {
            currentBuild.result = "FAILURE"
        }
        throw e
    } catch (InterruptedException e) {
        currentBuild.result = "ABORTED"
        throw e
    } catch (Throwable e) {
        currentBuild.result = "FAILURE"
        throw e
    } finally {
        stage("notifications") {
            //jenkinsNotify()
        }
    }
}

def buildProcess(String stageKey, String jdkName, String jdkTestName, String mvnName, goals, options, optionsIT, mavenOpts, boolean makeReports) {
    cleanWs()
    try {
        if (isUnix()) {
            sh 'mkdir -p .m2'
        } else {
            bat 'mkdir .m2'
        }
        def mvnLocalRepoDir = null
        dir('.m2') {
            mvnLocalRepoDir = "${pwd()}"
        }

        println "Maven Local Repository = ${mvnLocalRepoDir}."
        assert mvnLocalRepoDir != null : 'Local Maven Repository is undefined.'

        stage("checkout ${stageKey}") {
            checkout scm
        }

        def jdkTestHome = resolveToolNameToJavaPath(jdkTestName, mvnName)
        def script = ['mvn'] + goals + options

        stage("build ${stageKey}") {
            mvn(mvnName, script, mavenOpts, mvnLocalRepoDir, jdkName, jdkTestHome, makeReports)
            lock('maven-surefire-lock') {
                timeout(time: 2, unit: 'MINUTES') {
                    mvn(mvnName, script + optionsIT, mavenOpts, mvnLocalRepoDir, jdkName, jdkTestHome, false)
                }
            }
        }
    } finally {
        stage("reporting ${stageKey}") {
            if (makeReports) {
                openTasks(ignoreCase: true, canComputeNew: false, defaultEncoding: 'UTF-8', pattern: sourcesPatternCsv(),
                        high: tasksViolationHigh(), normal: tasksViolationNormal(), low: tasksViolationLow())

                jacoco(changeBuildStatus: false,
                        execPattern: '**/*.exec',
                        sourcePattern: sourcesPatternCsv(),
                        classPattern: classPatternCsv())

                junit(healthScaleFactor: 0.0,
                        allowEmptyResults: true,
                        keepLongStdio: true,
                        testResults: testReportsPatternCsv())

                if (currentBuild.result == 'UNSTABLE') {
                    currentBuild.result = 'FAILURE'
                }
            }

            if (fileExists('maven-failsafe-plugin/target/it')) {
                zip(zipFile: "it--maven-failsafe-plugin--${stageKey}.zip", dir: 'maven-failsafe-plugin/target/it', archive: true)
            }

            if (fileExists('surefire-its/target')) {
                zip(zipFile: "it--surefire-its--${stageKey}.zip", dir: 'surefire-its/target', archive: true)
            }

            archive includes: '*.zip'
        }

        stage("cleanup ${stageKey}") {
            // clean up after ourselves to reduce disk space
            cleanWs()
        }
    }
}

def mvn(mvnName, cmd, mavenOpts, mvnLocalRepoDir, jdkName, jdkTestHome, boolean makeReports) {
    withMaven(jdk: jdkName, maven: mvnName,
            mavenLocalRepo: mvnLocalRepoDir, mavenOpts: mavenOpts,
            options: [
                    findbugsPublisher(disabled: !makeReports),
                    openTasksPublisher(disabled: true),
                    junitPublisher(disabled: true),
                    artifactsPublisher(disabled: true),
                    invokerPublisher(disabled: true)
            ]) {
        def properties = ["\"-Djdk.home=${jdkTestHome}\"", "-Djacoco.skip=${!makeReports}"]
        println "Setting JDK for testing ${properties[0]}"
        cmd = cmd + properties
        if (isUnix()) {
            sh cmd.join(' ')
        } else {
            bat cmd.join(' ')
        }
    }
}

/**
 * It is used instead of tool(${jdkTestName}).
 */
def resolveToolNameToJavaPath(jdkToolName, mvnName) {
    def javaHome = null
    try {
        withMaven(jdk: jdkToolName, maven: mvnName) {
            //noinspection GrReassignedInClosureLocalVar
            javaHome = isUnix() ? sh(script: 'echo -en $JAVA_HOME', returnStdout: true) : bat(script: '@echo %JAVA_HOME%', returnStdout: true)
        }

        if (javaHome != null) {
            javaHome = javaHome.trim()
            def exec = javaHome + (isUnix() ? '/bin/java' : '\\bin\\java.exe')
            if (!fileExists(exec)) {
                println "The ${exec} does not exist in jdkToolName=${jdkToolName}."
                javaHome = null
            }
        }
    } catch(e) {
        println "Caught an exception while resolving 'jdkToolName' ${jdkToolName} via 'mvnName' ${mvnName}: ${e}"
        javaHome = null;
    }
    assert javaHome != null : "Could not resolve ${jdkToolName} to JAVA_HOME."
    return javaHome
}

@NonCPS
static def sourcesPatternCsv() {
    return '**/maven-failsafe-plugin/src/main/java,' +
            '**/maven-surefire-common/src/main/java,' +
            '**/maven-surefire-plugin/src/main/java,' +
            '**/maven-surefire-report-plugin/src/main/java,' +
            '**/surefire-api/src/main/java,' +
            '**/surefire-booter/src/main/java,' +
            '**/surefire-grouper/src/main/java,' +
            '**/surefire-its/src/main/java,' +
            '**/surefire-logger-api/src/main/java,' +
            '**/surefire-providers/**/src/main/java,' +
            '**/surefire-report-parser/src/main/java'
}

@NonCPS
static def classPatternCsv() {
    return '**/maven-failsafe-plugin/target/classes,' +
            '**/maven-surefire-common/target/classes,' +
            '**/maven-surefire-plugin/target/classes,' +
            '**/maven-surefire-report-plugin/target/classes,' +
            '**/surefire-api/target/classes,' +
            '**/surefire-booter/target/classes,' +
            '**/surefire-grouper/target/classes,' +
            '**/surefire-its/target/classes,' +
            '**/surefire-logger-api/target/classes,' +
            '**/surefire-providers/**/target/classes,' +
            '**/surefire-report-parser/target/classes'
}

@NonCPS
static def tasksViolationLow() {
    return '@SuppressWarnings'
}

@NonCPS
static def tasksViolationNormal() {
    return 'TODO,FIXME,@deprecated'
}

@NonCPS
static def tasksViolationHigh() {
    return 'finalize(),Locale.setDefault,TimeZone.setDefault,\
System.out,System.err,System.setOut,System.setErr,System.setIn,System.exit,System.gc,System.runFinalization,System.load'
}

@NonCPS
static def testReportsPatternCsv() {
    return '**/maven-failsafe-plugin/target/surefire-reports/*.xml,' +
            '**/maven-surefire-common/target/surefire-reports/*.xml,' +
            '**/maven-surefire-plugin/target/surefire-reports/*.xml,' +
            '**/maven-surefire-report-plugin/target/surefire-reports/*.xml,' +
            '**/surefire-api/target/surefire-reports/*.xml,' +
            '**/surefire-booter/target/surefire-reports/*.xml,' +
            '**/surefire-grouper/target/surefire-reports/*.xml,' +
            '**/surefire-its/target/surefire-reports/*.xml,' +
            '**/surefire-logger-api/target/surefire-reports/*.xml,' +
            '**/surefire-providers/**/target/surefire-reports/*.xml,' +
            '**/surefire-report-parser/target/surefire-reports/*.xml,' +
            '**/surefire-its/target/failsafe-reports/*.xml'
}
