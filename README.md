# LexaGene Build Server

This is a backup of LexaGene's build server's configuration.<br>
Contains:
- **/C/data/jenkins_home**
    - `Mainly the /*config/*.xml files; .gitignore the rest`
- **/C/data/jenkins_home/\_database**
    - `Local database build files; schema, tables and stored procedures.`
- **/C/data/jenkins_home/\_docs**
    - `e.g. /images/Build_Server_Diagram.PNG`
- **/C/data/jenkins_home/\_JenkinsConfigCopy**
    - `Copy of the server instance config that lives here: /C/tools/Jenkins/jenkins.xml`

> **About Build Numbers...**  
> The build number you see in Jenkins is not the build number for the artifact.  
> Artifcat build numbers are unique per REPO, and managed within build server MySQL database named lxg_versioning.  
> This makes guarantees versioning across branches and PRs is unique. Major.Minor.Patch.Build#**

## General Workflow
![](/_docs/images/Build_Server_Diagram.PNG)

## Login
- Build Server - //10.0.193.10 (must be within intranet/firewall, or through VPN)
  - User: lexagene_admin / Pwd: \<See BitWarden in the "Build Server" collection, named "BuildServerAdmin"\><br>**Jenkins service logs in as lexagene_admin.**
  - User: \<tim/rad/nathan\> / Pwd: \<user set\>

- Jenkins - http://localhost:8080/login (once logged into //10.0.193.10)
  - User: lexagene_admin / Pwd: \<build server password\>

- Build Server MySQL Database - lxg_versioning
  - User: "root" / Pwd: \<build server password\>  (the MySQL root/admin)
  - User: "lexagene_admin" / Pwd: \<empty\>

## Build Setup
1. **Create Multibranch Pipeline**<br>
(For each repository, create a new Multibranch Pipeline in Jenkins.)<br>
<a href="https://docs.cloudbees.com/docs/cloudbees-ci/latest/cloud-admin-guide/github-branch-source-plugin" target="_blank">Documentation: CloudBees GitHub Branch Source Plugin</a>
    - In Jenkins Dashboard, select "New Item"
        - Name = the repositories name. (e.g. "LeXml")
        - Select "Multibranch Pipeline"
        - Click OK
    - Add Branch Source
        - Add source = GitHub
        - Credentials = "Jenkins - LexaGene"
        - Repository HTTPS URL = <Copy/Paste from GitHub repo> (e.g. "https://github.com/LexaGene/LeXml.git")
        - Click Validate to test.
        - *Leave the rest default
    - Build Configuration
        - Mode = "by Jenkinsfile"
        - Script Path = "Jenkins/Jenkinsfile"
    - Click Save
    - *Repository will automatically scan for Jenkins files.*
2. **Jenkinsfile - Create and Configure**<br>
(For each branch you want built, add a Jenkins/"Jenkinsfile" to the repositories root directory)
    - Checkout the branch
    - Create folder "Jenkins" in the root directory.
    - Create file "Jenkinsfile" in the Jenkins folder. (No extention. Copy paste from another repo may help.)
    - **The build server will now try to execute the contents of this file every commit to this branch or open PR *to* this branch.**
3. **Build Number Helper**<br>
(LexaGene developed tool)
    - Add CALL_spBuildNumberGetNextAndIncr.ps1 to your Jenkins folder. (Copy this file from LeXml repo)
4. **Assembly Info Helper**<br>
(LexaGene developed tool)
    - *For C# applications such as LeXml, this updates the AssemblyVersion and AssemblyFileVersion info after chechout and before build.*
    - Add UpdateAssemblyInfo.ps1 to your Jenkins folder.
    - Modify this PowerShell script to operate on the AssemblyInfo.cs file you wish.
5. **How a Build is Initiated**
    - When the build server receives a commit or open PR notification from GitHub, and the branch has a  [Jenkins/Jenkinsfile], a build is initialized,

## Jenkinsfile Example/Workflow Stages
1. **Checkout**
    - Clean workspace
    - Checkout source from gitHub
2. **Get_Build_Number**
    - Query lxg_versioning database on build server for next build number, for the current repo.
3. **AssemblyInfoUpdate** (C# applications)
    - Update Properties/AssemblyInfo.cs files in applicable C# programs.
4. **Build**
    - For C#, typically:
        - nuget restore
        - Call MSBuild on solution file.
5. **Static Code Analysis**
6. **Run Tests**
7. **Archive**
    - Move build artifacts to an approprately accessible location off build server. 
    - **\\\\10.0.0.5\builds\\\<repo name>**

## Artifacts and Build File Locations
- Workspace - Where source files get checked out to, on the build server.
    - C:\data\jenkins_home\workspace\<RepoName>\_<Branch/PR Name>
- Build LOGS - The initial output building the source files.
    - C:\data\jenkins_home\jobs\<RepoName>\branches\<Branch/PR Name>\builds\<BuildNumber>
- Build OUTPUT (local)
    - C:\data\jenkins_home\jobs\<RepoName>\branches\<Branch/PR Name>\builds\<BuildNumber>\archive\*
- Build OUTPUT (external)
    - \\\\10.0.0.5\builds\\\<repo name>\\\<SementicVersion.BuildNumber>
    - **Only lexagene_admin user can modify/clean up. Read-only to all others.**
    - Artifact storage structure:

| Build type    | Artifact Structure |
| ------------- | ------------------ |
| Branch commit | \\\\10.0.0.5\\builds\\{REPOSITORY}\\{BRANCH_NAME}\\{Major}.{Minor}.{Patch}.{BuildNumber} |
| Open PR       | \\\\10.0.0.5\\builds\\{REPOSITORY}\\{TARGET_BRANCH_NAME}\\{Major}.{Minor}.{Patch}.{Build}-PR-{XXX} |

## Jenkins Installation Instructions
> See notes in \_Docs directory.
> 
> I followed this video:   https://www.jenkins.io/doc/book/installing/windows/

High level bullets:
- Java JDK - Install and configure environment.
- Local security policies - update.
- Jenkins - Install and configure environment.
  - **Runs as Windows Service**
- SMEE - Create reverse proxy for GitHub and Build Server to communicate through (so don't have to open server ports)
  - **Runs as Windows Scheduled Task, on system start up and forever**
- GitHub App - Create in GitHub, owned by LexaGene organization, and add to Jenkins Credentials.
- MySQL Database - Install and run Schema located in build-server repo at: [_Database]
- SAVE - Push Jenkins_Home directory to GitHub as new repo named LexaGene/build-server `https://github.com/LexaGene/build-server`

## Build Clean Up
The Jenkins plugin "GitHub Branch Source" cleans up the local workspace and jobs folders when branches are deleted.
> C:\data\jenkins_home\workspace  
> C:\data\jenkins_home\jobs\LeXml\branches  

**Cleaning up \\\\10.0.0.5\\builds:**<br>
The Jenkins plugin "MultiBranch Action Triggers" adds options to each Pipeline configuration to do things on certain triggers.<br>
I created a RunOnPipeline_Delete Freestyle project, that my Pipeline trigger executes on pipeline delete, and cleans up remote builds.
