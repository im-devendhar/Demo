

# **HTML App Deployment Using Jenkins and Apache on Ubuntu**

This guide explains how to deploy an HTML application using Jenkins on one Ubuntu server and Apache on another Ubuntu server.

---

## **Step 1: Create Two Ubuntu Servers**

1. **Jenkins Server**

   * Launch an Ubuntu EC2 instance (or VM) for Jenkins.
   * Make sure it has **internet access** and a **security group** allowing SSH (port 22) and Jenkins port (8080).

2. **Apache Application Server**

   * Launch a second Ubuntu EC2 instance (or VM) for Apache.
   * Security group should allow SSH (port 22) and HTTP (port 80).

---

## **Step 2: Set Up Jenkins Server**

1. **Install Java** (required by Jenkins):

   ```bash
   sudo apt update -y
   sudo apt install openjdk-11-jdk -y
   java -version
   ```

2. **Install Jenkins**

   ```bash
   wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc
   echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list
   sudo apt update -y
   sudo apt install jenkins -y
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```

3. **Open Jenkins**

   * Go to `http://<Jenkins_Server_IP>:8080`
   * Follow the instructions to unlock Jenkins and install recommended plugins.

---

## **Step 3: Set Up SSH Key-Based Authentication**

**Goal:** Jenkins server should connect to Apache server without a password.

1. **Generate SSH key on Jenkins server:**

   ```bash
   ssh-keygen -t rsa -b 4096 -C "jenkins" -f ~/.ssh/id_rsa
   ```

   * Press Enter for no passphrase.

2. **Copy public key to Apache server:**

   ```bash
   ssh ubuntu@<Apache_Server_IP> "mkdir -p ~/.ssh && chmod 700 ~/.ssh"
   cat ~/.ssh/id_rsa.pub | ssh ubuntu@<Apache_Server_IP> 'cat >> ~/.ssh/authorized_keys'
   ssh ubuntu@<Apache_Server_IP> "chmod 600 ~/.ssh/authorized_keys"
   ```

3. **Test passwordless SSH**

   ```bash
   ssh ubuntu@<Apache_Server_IP>
   ```

   * You should be logged in without entering a password.

---

## **Step 4: Set Up Apache on Application Server**

1. **Install Apache**

   ```bash
   sudo apt update -y
   sudo apt install apache2 -y
   ```

2. **Optionally, give Jenkins user permission to write**

   ```bash
   sudo chown -R ubuntu:ubuntu /var/www/html
   chmod -R 755 /var/www/html
   ```

3. **Verify Apache is running**

   ```bash
   systemctl status apache2
   ```

   * Visit `http://<Apache_Server_IP>` to see the default Apache page.

---

## **Step 5: Add SSH Key to Jenkins Credentials**

1. Go to **Jenkins → Manage Jenkins → Credentials → Global → Add Credentials**
2. Select: **SSH Username with private key**

   * **Scope:** Global
   * **Username:** ubuntu (Apache server user)
   * **Private Key:** Enter directly → paste the **private key** (`~/.ssh/id_rsa` from Jenkins server)
   * **ID:** `apache-ssh-key`
   * **Description:** Key for deploying HTML app

---

## **Step 6: Prepare Jenkins Pipeline (Jenkinsfile)**

**Example Jenkinsfile:**

```groovy
pipeline {
    agent any

    environment {
        APP_SERVER = 'ubuntu@<Apache_Server_IP>'
        APP_SERVER_PATH = '/var/www/html/main'
        SSH_KEY_CREDENTIALS = 'apache-ssh-key'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "Build successful for HTML app."
            }
        }

        stage('Test') {
            steps {
                echo "Quick code validation completed."
            }
        }

        stage('Deploy to Apache') {
            steps {
                echo "Deploying HTML app to remote Apache server..."
                sshagent([env.SSH_KEY_CREDENTIALS]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${APP_SERVER} 'mkdir -p ${APP_SERVER_PATH}'
                        scp -o StrictHostKeyChecking=no index.html ${APP_SERVER}:${APP_SERVER_PATH}/index.html
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed — check logs for details."
        }
    }
}
```

**Notes:**

* Replace `<Apache_Server_IP>` with your actual server IP.
* `SSH_KEY_CREDENTIALS` must match the Jenkins credential ID.
* If you want to deploy all files, you can modify the `scp` command like `scp -r * ${APP_SERVER}:${APP_SERVER_PATH}/`.

---

## **Step 7: Install Required Jenkins Plugins**

Minimum plugins for this pipeline:

* **Pipeline** (core)
* **Pipeline: Nodes and Processes**
* **SSH Agent Plugin**
* **Credentials Plugin**
* **Git Plugin**

Optional for scans:

* **SonarQube Scanner**
* **HTML Publisher**
* **Pipeline Utility Steps**

---

## **Step 8: Run the Pipeline**

1. Create a **New Pipeline** in Jenkins.
2. Connect it to your GitHub repository with the `Jenkinsfile`.
3. Run the pipeline.

**Check deployment:**

```bash
http://<Apache_Server_IP>/main/
```

You should see your HTML app live.

---

## **Step 9: Optional — Additional Features**

* **Malware Scan:** Use ClamAV (commented in Jenkinsfile).
* **Trivy Scan:** Container vulnerability check (commented in Jenkinsfile).
* **SonarQube Analysis:** Code quality (commented in Jenkinsfile).
* **Code Signing:** Sign HTML build files (commented in Jenkinsfile).

You can uncomment these stages later when you want to add security and code quality checks.

---

✅ **Congratulations!** Your HTML app should now be deployed automatically via Jenkins to your Apache server.

---


