# SonarQube and SonarCloud Setup Instructions

## Prerequisites
1. **Java**: Ensure you have Java version 11+ installed.
2. **Maven**: If you are using Maven, make sure to have it installed on your system.
3. **SonarQube Server**: Have access to a SonarQube server or create a free account on SonarCloud.
4. **Azure Account**: You will need an Azure account for setting up Azure Pipelines.

## Installation Steps
### SonarQube Installation
1. **Download SonarQube**: Visit [SonarQube Downloads](https://www.sonarqube.org/downloads/) and download the Community edition.
2. **Unzip and Start Server**: Unzip the downloaded file and go to the bin directory. Then, execute:
   ```bash
   ./sonar.sh start
   ```
3. **Access SonarQube**: Open your browser and go to `http://localhost:9000`.
4. **Login**: Use the default credentials (admin/admin) to log in.

### SonarCloud Setup
1. **Create Account**: Go to [SonarCloud](https://sonarcloud.io/) and create an account if you do not have one.
2. **Create a New Project**: Link your GitHub or Azure DevOps account and create a new project.
3. **Get Token**: Generate an access token under Account settings. This will be used for authentication.

## Azure Pipelines Integration
1. **Add SonarQube to Pipeline**: In your Azure Pipeline YAML file, add the following steps:
   ```yaml
   - task: SonarQubePrepare@4
     inputs:
       SonarQube: 'YourSonarQubeServiceConnection'
       scannerMode: 'CLI'
       configMode: 'manual'
       cliProjectKey: 'YourProjectKey'
       cliProjectName: 'YourProjectName'
   ```
2. **Run Analysis**: Add the analysis step in your YAML file:
   ```yaml
   - task: SonarQubeAnalyze@4
   ```
3. **Publish Quality Gate Result**: Add this step:
   ```yaml
   - task: SonarQubePublish@4
   ```

## Quality Gates Configuration
1. **Access Quality Gates**: In your SonarQube dashboard, navigate to 'Quality Gates' to configure your gates values.
2. **Set Conditions**: Define thresholds based on metrics like code coverage, code smells, bugs, and vulnerabilities.
   - Example: Set a minimum code coverage of 80%.

## Code Coverage Integration
1. **Enable Code Coverage**: In your pipeline YAML, ensure you have a task to generate code coverage reports. For example, with DotNet:
   ```yaml
   - script: dotnet test --collect:"XPlat Code Coverage"
   ```
2. **Add to SonarQube**: Make sure to include your coverage report in the SonarQube analysis step.

## Troubleshooting Guide
- **Common Issues**:
  - **Connection Issues**: Ensure SonarQube server is up and running.
  - **Authentication Issues**: Check your token and permissions.
  - **Quality Gate Failures**: Review configured metrics for possible reasons.

## Conclusion
Following these steps will help you set up SonarQube and integrate it with Azure Pipelines to maintain code quality. Ensure to regularly monitor your quality gates and fix identified issues promptly.
