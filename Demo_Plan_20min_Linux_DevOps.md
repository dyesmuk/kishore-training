# 20-Minute Demo: Linux (WSL) + Git + Jenkins + GitHub Actions

**One storyline:** a small Linux health-check script, written and secured on Linux, versioned with Git, built by Jenkins, and tested automatically by GitHub Actions.

| Time | Part | What the SMEs see |
|---|---|---|
| 0-1 min | Intro | The storyline and the repo |
| 1-6 min | Linux (WSL) | Permissions, ACL, logs, processes, cron |
| 6-9 min | Git | Commit, branch, push |
| 9-14 min | Jenkins | Pipeline: lint, test, package, archive |
| 14-18 min | GitHub Actions | Same checks on every push; one red-then-green run |
| 18-20 min | Wrap-up | Map to the ToC, labs, questions |

---

## A. Do this BEFORE the call (about 30 minutes; do not skip)

```bash
# 1. systemd on WSL (needed for journalctl)
cat /etc/wsl.conf            # should contain: [boot] systemd=true
# if not: add it, then in PowerShell: wsl --shutdown

# 2. Tools
sudo apt update
sudo apt install -y git acl openjdk-17-jdk curl

# 3. Jenkins (war file, no Docker needed)
mkdir -p ~/jenkins && cd ~/jenkins
curl -LO https://get.jenkins.io/war-stable/latest/jenkins.war
java -jar jenkins.war --httpPort=8080
# open http://localhost:8080 in Windows browser
# password: cat ~/.jenkins/secrets/initialAdminPassword
# install suggested plugins (takes several minutes), create admin user
# leave it running in its own terminal

# 4. GitHub
# create an empty repo (e.g. devops-demo) and sign in from WSL:
#   git config --global user.name "Vaman Deshmukh"
#   git config --global user.email "vamandeshmukh@gmail.com"
#   authenticate with a personal access token or `gh auth login`

# 5. Create the repo files below, commit once, and push so the first
#    GitHub Actions run is already green
```

### Repo files

**healthcheck.sh**
```bash
#!/bin/bash
# healthcheck.sh - warn when root disk usage crosses a threshold
THRESHOLD=${1:-80}
usage=$(df / --output=pcent | tail -1 | tr -dc '0-9')
if [ "$usage" -ge "$THRESHOLD" ]; then
  logger -t healthcheck -p user.warning "Disk usage ${usage}% >= ${THRESHOLD}%"
  echo "WARN: disk ${usage}%"
  exit 1
fi
logger -t healthcheck "Disk usage OK (${usage}%)"
echo "OK: disk ${usage}%"
```

**test.sh**
```bash
#!/bin/bash
set -e
./healthcheck.sh 100 >/dev/null            # threshold 100: must pass
if ./healthcheck.sh 0 >/dev/null; then     # threshold 0: must warn
  echo "FAIL: expected a warning"; exit 1
fi
echo "All tests passed"
```

**Jenkinsfile**
```groovy
pipeline {
  agent any
  stages {
    // Jenkins checks out the repo automatically when the job uses "Pipeline script from SCM"
    stage('Lint')    { steps { sh 'bash -n healthcheck.sh && echo "syntax OK"' } }
    stage('Test')    { steps { sh 'chmod +x *.sh && ./test.sh' } }
    stage('Package') {
      steps {
        sh 'tar czf healthcheck.tar.gz healthcheck.sh'
        archiveArtifacts artifacts: 'healthcheck.tar.gz'
      }
    }
  }
  post { always { echo "Build result: ${currentBuild.currentResult}" } }
}
```

**.github/workflows/ci.yml**
```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lint
        run: bash -n healthcheck.sh
      - name: Test
        run: chmod +x *.sh && ./test.sh
      - uses: actions/upload-artifact@v4
        with:
          name: healthcheck
          path: healthcheck.sh
```

**Jenkins job (create before the call):** New Item → Pipeline → *Pipeline script from SCM* → Git → your GitHub repo URL (public repo needs no credentials; a local path like `file:///home/<you>/devops-demo` also works because Jenkins runs on the same WSL machine) → branch `*/main` → Script Path `Jenkinsfile`. Run it once so it is green.

---

## B. Live script

### 1-6 min: Linux

```bash
cd ~/devops-demo
ls -l healthcheck.sh                 # note the permission bits
chmod 644 healthcheck.sh && ./healthcheck.sh   # Permission denied
chmod 755 healthcheck.sh && ./healthcheck.sh   # works
./healthcheck.sh 0; echo "exit=$?"             # warning, exit code 1

# logs: the script wrote to the journal
journalctl -t healthcheck -n 5 --no-pager

# ACL (one minute)
echo "release notes" > notes.txt && chmod 640 notes.txt
sudo adduser --disabled-password --gecos "" reviewer
setfacl -m u:reviewer:r notes.txt && getfacl notes.txt

# processes and jobs
sleep 300 &  jobs  ;  ps -ef | grep "[s]leep 300"  ;  kill %1
ps -ef | grep "[j]enkins.war"                  # Jenkins is just a Linux process
top -bn1 | head -5

# scheduling (show the line, do not wait for it)
echo "*/5 * * * * $HOME/devops-demo/healthcheck.sh"
```
Say: this is the day 1-2 flow (permissions, processes, users) and a taste of the logs/automation topics.

### 6-9 min: Git

```bash
git status
git switch -c feature/threshold-default
sed -i 's/THRESHOLD=${1:-80}/THRESHOLD=${1:-85}/' healthcheck.sh
git diff
git commit -am "Raise default disk threshold to 85"
git switch main && git merge feature/threshold-default
git log --oneline --graph -5
git push origin main
```
Say: branch, diff, commit, merge, push. Mention pull requests and branch protection as the team workflow.

### 9-14 min: Jenkins

1. Open `http://localhost:8080` → your job → **Build Now** (the push from Git triggers it only if you enabled polling; manual is fine for the demo).
2. Open the build → **Pipeline Steps / Stage View** → show Lint, Test, Package.
3. Show **Console Output** (it is the same `./test.sh` you ran by hand) and the archived `healthcheck.tar.gz`.
4. Say: `agent any` runs on this Linux machine; in a real setup you would use agents; a Jenkinsfile lives in the repo so the pipeline is versioned too.

### 14-18 min: GitHub Actions

1. Show the **Actions** tab with the green run from your last push (workflow file, runner, steps).
2. Break something on purpose and push:
   ```bash
   sed -i 's/exit 1/exit 0/' healthcheck.sh
   git commit -am "Introduce bug" && git push
   ```
   Show the red run and the failing step.
3. Fix it:
   ```bash
   git revert --no-edit HEAD && git push
   ```
   Show it turning green.
4. Say: Jenkins vs GitHub Actions: Jenkins is self-hosted with a large plugin ecosystem; GitHub Actions is hosted and tied to the repo. Many teams use one or both.

### 18-20 min: Wrap-up

- Linux demo ties to the ToC days on permissions, processes, logs and automation. Full storage/LVM, Pacemaker and the RHEL upgrade are done on VM labs.
- DevOps demo ties to days 2-3 (Git, Jenkins, GitHub Actions); Docker/Kubernetes, Terraform and SRE are covered in the remaining days.
- Mention the lab setup you would need and ask the questions from the prep sheet.

---

## C. If something breaks during the call

- **Jenkins not loading or slow:** show the earlier green build and the Jenkinsfile; describe the stages.
- **GitHub push or Actions delayed:** show the previous green run, then say the red/green flow takes under a minute normally.
- **`journalctl` empty:** systemd isn't enabled; run `logger` and `tail` on `/var/log/syslog` if it exists, or skip to the permission demo.
- **Do not troubleshoot for more than 30 seconds on camera.** Move on and mention the fallback.
