# Synthetic Monitoring with Groovy: Git Operations and YAML Processing

## Overview

This Groovy script is designed to perform synthetic monitoring tasks by automating the process of cloning or updating a Git repository, reading a YAML configuration file, and preparing HTTP request data for further processing. The script handles proxy settings, ensures the repository is up-to-date, and processes YAML data to generate a list of HTTP requests based on the configuration. This setup is particularly useful for environments where monitoring tasks need to be dynamically configured based on changes in a central repository.

## Prerequisites

Before running this script, ensure the following:

- **Git**: Installed and configured on the system where the script will be executed.
- **Proxy Configuration**: If your environment requires a proxy, configure the proxy server and port in the script.
- **Access Token**: A valid access token with sufficient permissions to clone and fetch the Git repository.
- **YAML Configuration**: A YAML file located in the specified directory that contains the configuration for the monitoring tasks.
- **Groovy Environment**: The script is executed in an environment that supports Groovy scripting, such as JMeter with JSR223 Sampler.

## Script Workflow

1. **Proxy Setup**:
  - Configures the global Git proxy settings to ensure Git commands can execute properly in environments requiring a proxy.

2. **Git Operations**:
  - Checks if the local directory for the repository exists.
  - If the directory does not exist, the script clones the repository using an authenticated URL.
  - If the directory exists, the script updates the repository by fetching the latest changes and resetting the working directory to match the remote `main` branch.

3. **YAML Processing**:
  - Once the repository is updated or cloned, the script reads the YAML configuration file.
  - The YAML content is parsed, and the script processes the configuration to generate a list of HTTP requests based on active modules, hosts, and load balancers.

4. **Data Preparation**:
  - The script merges default settings from the YAML with specific entity properties, ensuring that the entity's own properties take precedence.
  - The processed data is stored in a JMeter variable (`vars`) for use in subsequent test steps or samplers.

5. **Cleanup**:
  - Resets the Git proxy settings to avoid affecting future Git operations outside the script's scope.

## Script Code

```groovy
import org.yaml.snakeyaml.Yaml
import java.io.File

def executeCommand(String command) {
   def process = command.execute()
   process.waitFor()
   def output = process.in.text.trim()
   def error = process.err.text.trim()
   return [output: output, error: error, exitCode: process.exitValue()]
}

// Replace with your proxy server and port if needed
String proxyServer = "your.proxy.server"
int proxyPort = 3128
executeCommand("git config --global http.proxy http://${proxyServer}:${proxyPort}")
executeCommand("git config --global https.proxy https://${proxyServer}:${proxyPort}")

// Local directory and repository information
String localDir = "E:/scripts/project/Observability"
// To switch source repo feel free to change the below repoURL variable.
String repoURL = "https://github.com/your-organization/your-repo.git"
String accessToken = "your_access_token"
String yamlFile = localDir + "/path/to/your.yaml"

// Correctly format the URL with the access token for cloning and fetching
String authenticatedRepoURL = repoURL.replaceFirst("https://", "https://" + accessToken + "@")
File dir = new File(localDir)

if (!dir.exists()) {
   // Use the authenticatedRepoURL with the token embedded for operations
   def cloneResult = executeCommand("git clone ${authenticatedRepoURL} ${localDir}")
   log.info("Git Clone Result: ${cloneResult.output}, Error: ${cloneResult.error}, Exit Code: ${cloneResult.exitCode}")
} else {
   // Update the remote URL to the authenticated one before fetching
   executeCommand("git -C ${localDir} remote set-url origin ${authenticatedRepoURL}")
   def fetchResult = executeCommand("git -C ${localDir} fetch origin")
   log.info("Git Fetch Result: ${fetchResult.output}, Error: ${fetchResult.error}, Exit Code: ${fetchResult.exitCode}")
   
   def resetResult = executeCommand("git -C ${localDir} reset --hard origin/main")
   log.info("Git Reset to Remote HEAD Result: ${resetResult.output}, Error: ${resetResult.error}, Exit Code: ${resetResult.exitCode}")
}

// After ensuring the repo is updated or cloned, read and process the YAML
try {
   String yamlContent = new File(yamlFile).getText('UTF-8')
   Yaml yaml = new Yaml()
   Map<String, Object> data = yaml.load(yamlContent)
   List<Map<String, String>> httpRequestData = []
   data.eais.each { eai ->
       eai.modules.each { module ->
           if (module.active != false) {  // Only process active modules
               def defaults = module.defaults
               def entities = []
               if (module.hosts) {
                   module.hosts.each { host ->
                       if (host.active != false) {  // Only add active hosts
                           entities << host
                       }
                   }
               }
               if (module.load_balancers) {
                   module.load_balancers.each { lb ->
                       if (lb.active != false) {  // Only add active load balancers
                           entities << lb
                       }
                   }
               }
               entities.each { entity ->
                   // Merge entity properties with defaults; ensure entity's own properties take precedence
                   def mergedEntity = defaults + entity
                   httpRequestData.add(mergedEntity + ['EAI': eai.eai, 'Module': module.module_name])
               }
           }
       }
   }
   vars.putObject("httpRequestData", httpRequestData)
   log.info("Output to console: ${vars.get('httpRequestData')}")
} catch (Exception e) {
   log.error("Error processing YAML or populating HTTP request data: ${e.message}", e)
}

// Reset proxy configuration to avoid affecting future Git operations
executeCommand("git config --global --unset http.proxy")
executeCommand("git config --global --unset https.proxy")
```

## Detailed Explanation

### 1. **Proxy Setup**
  - Configures Git to use a proxy server for both HTTP and HTTPS requests. This ensures that the script can interact with remote repositories in environments that require proxy settings.

  ```groovy
  executeCommand("git config --global http.proxy http://${proxyServer}:${proxyPort}")
  executeCommand("git config --global https.proxy https://${proxyServer}:${proxyPort}")
  ```

### 2. **Git Operations**
  - **Cloning the Repository**: If the target directory does not exist, the script clones the repository using an authenticated URL.
  - **Fetching Updates**: If the directory exists, the script fetches the latest changes and resets the local branch to match the remote `main` branch.

  ```groovy
  if (!dir.exists()) {
      def cloneResult = executeCommand("git clone ${authenticatedRepoURL} ${localDir}")
      log.info("Git Clone Result: ${cloneResult.output}, Error: ${cloneResult.error}, Exit Code: ${cloneResult.exitCode}")
  } else {
      executeCommand("git -C ${localDir} remote set-url origin ${authenticatedRepoURL}")
      def fetchResult = executeCommand("git -C ${localDir} fetch origin")
      log.info("Git Fetch Result: ${fetchResult.output}, Error: ${fetchResult.error}, Exit Code: ${fetchResult.exitCode}")
      
      def resetResult = executeCommand("git -C ${localDir} reset --hard origin/main")
      log.info("Git Reset to Remote HEAD Result: ${resetResult.output}, Error: ${resetResult.error}, Exit Code: ${resetResult.exitCode}")
  }
  ```

### 3. **YAML Processing**
  - The script reads a YAML file that contains configuration data for various monitoring tasks. It processes this data, filtering out inactive modules and merging entity-specific settings with default values.

  ```groovy
  String yamlContent = new File(yamlFile).getText('UTF-8')
  Yaml yaml = new Yaml()
  Map<String, Object> data = yaml.load(yamlContent)
  ```

### 4. **Data Preparation**
  - Processes the YAML data to generate a list of HTTP request data objects. These objects are stored in the JMeter variable `httpRequestData` for use in subsequent requests or tests.

  ```groovy
  vars.putObject("httpRequestData", httpRequestData)
  log.info("Output to console: ${vars.get('httpRequestData')}")
  ```

### 5. **Cleanup**
  - Resets the Git proxy settings after the script finishes executing, ensuring that future Git operations are not affected by the proxy configuration used in this script.

  ```groovy
  executeCommand("git config --global --unset http.proxy")
  executeCommand("git config --global --unset https.proxy")
  ```

## Usage and Execution

- **Integration with prometheus/influx**:  
 This script can be executed within a Groovy environment to dynamically configure and execute synthetic monitoring tasks based on the latest configurations from a central Git repository.

- **Dynamic Configuration**:  
 By pulling the latest configuration from a Git repository, the script ensures that the monitoring tasks are always up-to-date and reflect the current state of the system being monitored.
