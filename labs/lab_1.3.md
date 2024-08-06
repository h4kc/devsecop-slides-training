# LAB 1.3

## Automate Static Analysis

### Goals

- Perform a local Static Security testing
- Add a SAST stage in jenkins
- Adjust jenkins thresholds

Clone the **express-api** from gitlab

let’s discover semgrep rules 

Go to [https://semgrep.dev/](https://semgrep.dev/)

Discover the registry 

Discover the playground

 create a folder called semgrep and create a Docker file for the semgrep image

```bash
# Use an official Python runtime as a parent image
FROM python:3.9-slim
USER root

# Install Semgrep
RUN pip install semgrep

# Set the working directory
WORKDIR /app
ENTRYPOINT []
```

build the image

```bash
cd semgrep && docker build -t semgrep-express . 
```

check which version of semgrep  is installed 

```bash
docker run --rm  semgrep-express semgrep --version
```

Run the scan locally

```bash
docker run --rm -v .:/app semgrep-express semgrep --config p/expressjs
```

analyze the findings 

let’s move to the CI

```bash
pipeline {
    agent none
    stages {
        stage('Semgrep Builder') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    dir 'semgrep'
                    additionalBuildArgs '--tag devsecops/semgrep --network host'
                } }
            steps {
                sh 'echo "SEMGREP INSTALLED"'
            }
            }
        stage('SAST') {
            agent {
                docker {
                    image 'devsecops/semgrep'
                    args '-u root --privileged --network host'
            }
        }
            steps {
                sh 'semgrep --version'
                sh "semgrep --config p/expressjs"
            }
        }
        stage('Dependancy checking') {
            agent any
            steps {
                sh 'echo Dependancy checking'
            }
        }
    }
}

```

let’s save the scan report 

```bash
pipeline {
    agent none
    stages {
        stage('Semgrep Builder') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    dir 'semgrep'
                    additionalBuildArgs '--tag devsecops/semgrep --network host'
                } }
            steps {
                sh 'echo "SEMGREP INSTALLED"'
            }
            }
        stage('SAST') {
            agent {
                docker {
                    image 'devsecops/semgrep'
                    args '-u root --privileged --network host'
            }
        }
            steps {
                sh 'semgrep --version'
                sh "semgrep --config p/expressjs --junit-xml -o ${env.WORKSPACE}/semgrep.junit.xml"
                archiveArtifacts 'semgrep.junit.xml'

            }
        }
        stage('Dependancy checking') {
            agent any
            steps {
                sh 'echo Dependancy checking'
            }
        }
    }
}

```

let’s define a Threshold

```bash
pipeline {
    agent none
    stages {
        stage('Semgrep Builder') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    dir 'semgrep'
                    additionalBuildArgs '--tag devsecops/semgrep --network host'
                } }
            steps {
                sh 'echo "SEMGREP INSTALLED"'
            }
            }
        stage('SAST') {
            agent {
                docker {
                    image 'devsecops/semgrep'
                    args '-u root --privileged --network host'
            }
        }
            steps {
                sh 'semgrep --version'
                sh "semgrep --config p/expressjs --junit-xml -o ${env.WORKSPACE}/semgrep.junit.xml"
                archiveArtifacts 'semgrep.junit.xml'
                xunit(
                    tools: [GoogleTest(pattern : 'semgrep.junit.xml')], 
                    thresholds: [failed(failureThreshold: '100')],
                    )
            }
        }
        stage('Dependancy checking') {
            agent any
            steps {
                sh 'echo Dependancy checking'
            }
        }
    }
}

```
Dependency checking ---- by m.a

```bash
pipeline {
    agent none
    stages {
        stage('Semgrep Builder') {
            agent {
                dockerfile {
                    filename 'Dockerfile'
                    dir 'semgrep'
                    additionalBuildArgs '--tag devsecops/semgrep --network host'
                } }
            steps {
                sh 'echo "SEMGREP INSTALLED"'
            }
            }
        stage('SAST') {
            agent {
                docker {
                    image 'devsecops/semgrep'
                    args '-u root --privileged --network host'
            }
        }
            steps {
                sh 'semgrep --version'
                sh "semgrep --config p/expressjs --junit-xml -o ${env.WORKSPACE}/semgrep.junit.xml"
                archiveArtifacts 'semgrep.junit.xml'
                xunit(
                    tools: [GoogleTest(pattern : 'semgrep.junit.xml')], 
                    thresholds: [failed(failureThreshold: '100')],
                    )
            }
        }
        stage('OWASP Dependency-Check Vulnerabilities') {
        agent any
      steps {
        dependencyCheck additionalArguments: ''' 
                    -o './'
                    -s './'
                    -f 'ALL' 
                    --prettyPrint''', odcInstallation: 'OWASP Dependency-Check Vulnerabilities'
        
        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
      }
    }
    }
}

```
