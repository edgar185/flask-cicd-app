# Flask CI/CD Pipeline with Jenkins, AWS CodeBuild, and AWS CodeDeploy

An end-to-end continuous integration and continuous deployment pipeline that automatically builds, tests, and deploys a Flask web application to EC2 instances on every commit to `main`.

![Status](https://img.shields.io/badge/status-deployed-success)
![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20S3%20%7C%20CodeBuild%20%7C%20CodeDeploy-orange)
![Jenkins](https://img.shields.io/badge/Jenkins-LTS-blue)
![Python](https://img.shields.io/badge/Python-3.x-yellow)

---

## Architecture

```
GitHub  ──▶  Jenkins (Poll SCM)  ──▶  AWS CodeBuild  ──▶  S3 (Artifact Store)
                                                                  │
                                                                  ▼
                                              AWS CodeDeploy  ──▶  EC2 (App Servers)
```

Jenkins acts as the central orchestrator, while AWS services handle compute, storage, and deployment. The pipeline is triggered automatically whenever a new commit lands on the `main` branch.

---

## How It Works

| Stage | Service | Action |
|-------|---------|--------|
| 1. Detect | Jenkins (Poll SCM) | Checks GitHub every 2 minutes for new commits |
| 2. Build | AWS CodeBuild | Installs dependencies, runs unit tests, packages artifact |
| 3. Store | Amazon S3 | Stores the build artifact (`codebuild-artifact.zip`) |
| 4. Deploy | AWS CodeDeploy | In-place deployment across tagged EC2 instances |
| 5. Run | EC2 (Amazon Linux 2023) | Flask application serves on port 80 |

---

## Repository Contents

```
flask-cicd-app/
├── web.py                  # Flask application entry point
├── test_app.py             # Unit tests (run by CodeBuild)
├── requirements.txt        # Python dependencies
├── buildspec.yml           # AWS CodeBuild build instructions
├── appspec.yml             # AWS CodeDeploy lifecycle hooks
├── templates/              # Flask HTML templates
│   ├── layout.html
│   └── test.html
└── scripts/                # CodeDeploy lifecycle scripts
    ├── mkdir.sh            # AfterInstall: creates /web directory
    ├── start_flask.sh      # ApplicationStart: launches Flask
    ├── stop_flask1.sh      # ApplicationStop: gracefully stops app
    └── stop_flask.py       # Helper: sends shutdown request
```

---

## Prerequisites

To replicate this pipeline, you'll need:

- An AWS account with permissions to create IAM roles, EC2 instances, S3 buckets, CodeBuild projects, and CodeDeploy applications
- A GitHub account with a personal access token (scoped to `repo`)
- Local tools: `git`, a terminal, and a web browser
- Approximate AWS cost: t3.large (Jenkins) + 2x t3.small (app servers) running for the duration of the exercise

---

## Setup Overview

The full setup involves these high-level steps. Each is documented in detail in the original course material — this repo provides the application code and configuration that the pipeline deploys.

1. **Create GitHub repository** and push the application code
2. **Create IAM roles** — `JenkinsServerRole`, `CodeBuildServiceRole`, `CodeDeployServiceRole`, `CodeDeployInstanceRole`
3. **Launch Jenkins EC2 instance** with Java 21 and Jenkins LTS installed via user data
4. **Create S3 bucket** for build artifacts with an IP-restricted bucket policy
5. **Create CodeBuild project** pointing to the GitHub repo, using `buildspec.yml`
6. **Launch app server EC2 instances** with the CodeDeploy agent installed and `Environment=Production` tag
7. **Create CodeDeploy application and deployment group** targeting tagged instances
8. **Install Jenkins plugins**: AWS CodeBuild, AWS CodeDeploy, File Operations, HTTP Request
9. **Configure Jenkins credentials** with AWS access keys
10. **Create Jenkins freestyle project** wiring everything together with build steps and post-build action

Once configured, every push to `main` triggers the full pipeline automatically.

---

## Key Technical Decisions

**Why Jenkins over native AWS CodePipeline?**
Jenkins offers cloud-agnostic orchestration. The same pipeline pattern can be ported to GCP, Azure, or on-prem environments by swapping plugins. AWS services do the heavy lifting; Jenkins owns the workflow logic.

**Why in-place deployment over blue/green?**
Simpler for a learning exercise and lower cost. For production, blue/green via CodeDeploy + an Application Load Balancer would be the next step.

**Why Poll SCM over webhooks?**
Polling avoids opening Jenkins to inbound traffic from GitHub. For higher-velocity teams, GitHub webhooks would reduce latency from ~2 minutes to seconds.

---

## Troubleshooting Notes

A few non-obvious issues encountered during setup, documented here for anyone replicating this in 2026 or later:

### Java version
Current Jenkins LTS requires Java 21+. Tutorials still referencing Java 17 will fail with:
```
Running with Java 17 ... which is older than the minimum required version (Java 21)
```
Fix: install `java-21-amazon-corretto-headless` and update the systemd `JAVA_HOME` override.

### CodeBuild Jenkins plugin UI
Recent versions require selecting **Use Project source** via radio button (not the legacy "Source Type" dropdown). The error `Source control type is required and must be 'jenkins' or 'project'` indicates this.

### Python version on Amazon Linux 2023
AL2023 ships with `python3` only — no `python` symlink, no `python3.7`. Lifecycle scripts written for older Amazon Linux 2 environments need updating.

### CodeDeploy state caching
A failed deployment caches its scripts at `/opt/codedeploy-agent/deployment-root/`. The agent will run the *previous* deployment's `ApplicationStop` script before downloading a new bundle, which can deadlock the pipeline. Clear the archive and restart the agent to recover:
```bash
sudo rm -rf /opt/codedeploy-agent/deployment-root/*
sudo systemctl restart codedeploy-agent
```

### File collision protection
CodeDeploy refuses to overwrite existing files at the target path by default. Either enable "Overwrite the content" in the deployment group settings, or clean the target directory before redeployment.

---

## What I Learned

- Reading CodeDeploy lifecycle event logs is the fastest path to diagnosing deployment failures
- Tutorial documentation ages faster than the underlying tools — Java versions, plugin UIs, and Linux defaults all shift
- Jenkins's flexibility comes with ongoing plugin compatibility maintenance
- IAM scoping matters: four distinct roles isolate Jenkins, build, deploy service, and deploy instance permissions

---

## Cleanup

To avoid AWS charges after testing, delete resources in reverse order:

1. Terminate EC2 instances (Jenkins-DevOps, App-Server-1, App-Server-2)
2. Delete CodeDeploy application and deployment group
3. Delete CodeBuild project
4. Empty and delete the S3 bucket
5. Delete IAM roles
6. Delete security groups
7. **Delete any AWS access keys created for Jenkins** — these are particularly risky to leave behind

---

## Acknowledgments

Built as part of the Great Learning PGP in Cloud Computing program. The application skeleton (Flask app + lifecycle scripts) was provided as course material; the pipeline configuration, troubleshooting, and AL2023/Java 21 modernization are my own work.

---

## License

This project is for educational purposes. Feel free to fork and adapt.
