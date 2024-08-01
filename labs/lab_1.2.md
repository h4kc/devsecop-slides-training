# LAB 1.2

## Version Control Security

### Goals

- Protect main branch
- Separation of duties
- Security checklist in the pull request template
- Mandatory code reviews
- Code owners
- pre-commit hooks

### Launch Gitea using docker-compose

Create docker-compose.yml

```docker
version: "3"
networks:
  gitea:
    external: false
services:
  server:
    image: gitea/gitea:latest
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "222:22"
```

Execute the docker compose command

```bash
docker compose up
```

Configure, install gitea and add the admin user

Login as admin

Create users : jad, ahmed, hind, yassine

Create an organization :  devsecops-training-lab

In the organization create teams : admin, devops, security, coder

Add each user to a team

Create a repository inside the organization

Push any code that have Jenkinsfile and Dockerfile

Configure branch protection (Disable push, assign merge to the admin team, set the number of approval to 2)

Add CODEOWNER file

```markdown
# Owners for Jenkinsfile
Jenkinsfile @devsecops-training-lab/security @devsecops-training-lab/devops

# Owners for Dockerfile
Dockerfile @devsecops-training-lab/devops

```

Add a pull request checklist template

```markdown
## Description
<!-- Please include a summary of the changes and the related issue. -->
## Checklist

### General
- [ ] I have performed a self-review of my own code.
- [ ] I have commented my code, particularly in hard-to-understand areas.
- [ ] I have made corresponding changes to the documentation.
- [ ] My changes generate no new warnings.
### Testing
- [ ] New and existing unit tests pass locally with my changes.
- [ ] I have checked my code and corrected any misspellings.
### Security
- [ ] No sensitive information (like credentials) are included in this PR.
- [ ] All the users iput are validated.
- [ ] I checked access controle issues.
- [ ] New and existing secutity unit tests pass locally with my changes.
```

Play with the gitflow

### Pre-commit Hooks

first install pre-commit

```bash
 sudo apt install pre-commit
```

add the config file : .pre-commit-config.yaml 

```yaml
repos:
  - repo: https://github.com/zricethezav/gitleaks
    rev: v8.2.7 # use the latest version available
    hooks:
      - id: gitleaks
  - repo: local
    hooks:
      - id: detect-console-log
        name: Detect console.log
        entry: ./console-log-detecter.sh
        language: system
        types: [javascript]
```

create the script that will be executed after the git commit command ./console-log-detecter.sh, this script will detect if there is any console.log in he code

```bash
#!/bin/sh
# This script checks for console.log statements in .js files, excluding node_modules

# Find .js files and search for console.log, excluding node_modules directory
grep -r -n --exclude-dir="node_modules" --include="*.js" "console.log" .

# Exit with status 1 if any console.log is found
if [ $? -eq 0 ]; then
    echo "console.log found. Please remove it before committing."
    exit 1
fi

```

run  chmod +x

```bash
 chmod +x console-log-detecter.sh 
```

add suspicipcious lines to the app.js

```bash
// This is a vulnerable example with a fake secret
const apiKey = "1234567890abcdef1234567890abcdef";
console.log("This is a test file");

```

run the git add and the git commit command.