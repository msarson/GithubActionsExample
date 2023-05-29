This repository contains an example YML file (pronounced YAML) for build processes needed to generate your builds of a
Clarion Application. 


There are various steps here that can be removed if not required. I am using UpperPark source control from Rick Martin
as part of this eample.

**Things you will need to do and take note of**
* You will need a private repository that contains the build of clarion you want to use.
* The yaml file you generate will go into your dev repository in a directory called .github\workflows
* YML files are very specific as to where the columns of each line should be, so please follow the example.
**NOTE** GITHUB_SECRET is a Personal Access Token you create, and add to the dev repository, name however you want.

This explanation will update as I can.
```yaml
name: Clarion Build

on:
  push:
    branches:
      # Branches that this action will run, probably not a good idea on master :)
      - master
```      
This YAML configuration sets the name of the GitHub Action to "Clarion Build" and specifies that it should run when a push event occurs on the specified branches (in this case, only the master branch).
```yaml          

jobs:
  build:
    runs-on: windows-latest
```
This defines the build job, which will run on the latest version of the Windows operating system. The steps section contains the individual steps that will be executed as part of the job.
```yaml
    steps:
      - name: Install Git LFS
        # Install Git LFS and display version information
        run: |
          git lfs install --skip-repo
          git lfs version
```          
This step installs Git LFS (Large File Storage) and displays its version information.
```yaml
      - name: Checkout code
        # Checkout the repository's code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
          lfs: true
```
This step checks out the repository's code using the actions/checkout action. It sets the fetch-depth option to 0 to fetch all history and enables Git LFS for large file handling.
```yaml
      - name: Pull from Private Repository
        # Pull clarion from a private repository into the 'clarion' directory
        uses: actions/checkout@v3
        with:
          repository: username/RepositoryName
          path: clarion
          ref: master
          token: ${{ secrets.GITHUB_SECRET }}
          lfs: true
```
This step pulls the clarion repository **Its important to keep this repository private** and places it in the clarion directory in this repository. It uses the actions/checkout action, specifies the master branch (ref), and provides a token for authentication (token).
```yaml
      - name: Modify ClarionProperties.xml
        # Modify ClarionProperties.xml file to replace a specific path
        run: |
            $clarionPropertiesPath = "${{ github.workspace }}/appdata/ClarionProperties.xml"
            $backupPath = "${{ github.workspace }}/appdata/ClarionProperties.xml.bak"
        
            # Backup ClarionProperties.xml
            Copy-Item -Path $clarionPropertiesPath -Destination $backupPath -Force
        
            # Read the contents of the ClarionProperties.xml file
            $clarionPropertiesContent = Get-Content -Path $clarionPropertiesPath
        
            # Define the replacement value
            $replacement = "$($env:GITHUB_WORKSPACE)\clarion"
        
            # Replace references to C:\clarion\clarion11 with $env:GITHUB_WORKSPACE path
            $modifiedContent = $clarionPropertiesContent -replace 'C:\\clarion\\clarion11', $replacement
        
            # Save the modified content back to the ClarionProperties.xml file
            $modifiedContent > $clarionPropertiesPath
        
            # Copy the backup file back to the original path
            Copy-Item -Path $clarionPropertiesPath -Destination $backupPath -Force
        
            git config --global user.name "GitHub Actions"
            git config --global user.email "actions@github.com"

            # Add the modified file to the git index
            git add $backupPath
        
            # Commit the changes with a specific message
            git commit -m "Rebackup of ClarionProperties.xml on failure"
        
            # Push the changes to the repository
            git push
```
You're dev directory will have a folder called appdata where you copy your clarion properties (AppData\Roaming\SoftVelocity\Clarion\VersionNumber) to. This allows the process to use your licence for the build.

This step modifies the ClarionProperties.xml file by replacing specific path references. It creates a backup of the original file, reads its contents, performs the replacement, and saves the modified content. It then configures Git user information, adds the modified file to the git index, commits the changes, and pushes them to the repository.
```yaml
      - name: Setup .NET Framework 4.6
        # Install .NET Framework 4.6 using Chocolatey
        run: |
          choco install dotnet4.6 --ignore-checksums -y
          refreshenv
```
This step installs .NET Framework 4.6 using Chocolatey, a package manager for Windows. It then refreshes the environment variables.
```yaml
      - name: Setup MSBuild
        # Setup MSBuild for building the project
        uses: microsoft/setup-msbuild@v1.1
```
This step sets up MSBuild, the Microsoft Build Engine, which is used for building the project. It uses the microsoft/setup-msbuild action provided by Microsoft.
```yaml
      - name: Generate txa files, they will be named UPS
        # Generate txa files using ClaInterface.exe
        run: |
          $claInterfacePath = "${{ github.workspace }}\clarion\clainterface\ClaInterface.exe"
          $arguments = "COMMAND=BUILDAPP INPUT=${{ github.workspace }}\vcTestWorkflow  OUTPUT=${{ github.workspace }}"
          # Start the process in the background
          Start-Process -FilePath $claInterfacePath -ArgumentList $arguments -NoNewWindow -PassThru
          # Wait for the process to complete
          Wait-Process -Name ClaInterface
```
This step generates txa files using ClaInterface.exe. It sets the necessary command-line arguments, starts the process in the background, and waits for it to complete. 
```yaml
      - name: Import txa files, only way to do this is list them
        # Import txa files into the project
        run: |
          ${{ github.workspace }}/clarion/bin/clarioncl.exe  /ConfigDir "${{ github.workspace }}/appdata" /up_createappVC ${{ github.workspace }}\Helloworld.sln ${{ github.workspace }}\data.ups data
          ${{ github.workspace }}/clarion/bin/clarioncl.exe  /ConfigDir "${{ github.workspace }}/appdata" /up_createappVC ${{ github.workspace }}\Helloworld.sln ${{ github.workspace }}\HelloWorld.ups HelloWorld
```
This step imports the txa files into the project using clarioncl.exe. It specifies the necessary command-line arguments for importing the files.
```yaml
      - name: Generate Applications
        # Generate applications using Clarion's clarioncl.exe
        run: |
          ${{ github.workspace }}/clarion/bin/clarioncl.exe  /ConfigDir "${{ github.workspace }}\appdata" /ag "${{ github.workspace }}\Helloworld.sln"
```
This step generates applications using clarioncl.exe. It specifies the necessary command-line arguments for generating the applications.
```yaml
      - name: Build Clarion project
        # Build Clarion project using MSBuild
        run: |
          msbuild data.cwproj `
            /property:GenerateFullPaths=true `
            /t:build `
            /m `
            /consoleloggerparameters:NoSummary `
            /property:Configuration=Debug `
            /property:clarion_Sections=Debug `
            /property:SolutionDir=${{ github.workspace }} `
            /property:ClarionBinPath="Clarion\Bin" `
            /property:NoDependency=true `
            /property:Verbosity=diagnostic `
            /property:WarningLevel=1 `

          msbuild HelloWorld.cwproj `
            /property:GenerateFullPaths=true `
            /t:build `
            /m `
            /consoleloggerparameters:NoSummary `
            /property:Configuration=Debug `
            /property:clarion_Sections=Debug `
            /property:SolutionDir=${{ github.workspace }} `
            /property:ClarionBinPath="Clarion\Bin" `
            /property:NoDependency=true `
            /property:Verbosity=diagnostic `
            /property:WarningLevel=1 `
```
This step builds the Clarion project using MSBuild. It runs two separate MSBuild commands, one for data.cwproj and another for HelloWorld.cwproj, with various command-line arguments for configuration and customization. you will need to replace with your various parts.
```yaml
      - name: Restore ClarionProperties.xml
        # Restore the original ClarionProperties.xml file
        run: |
            $clarionPropertiesPath = "${{ github.workspace }}/appdata/ClarionProperties.xml"
            $backupPath = "${{ github.workspace }}/appdata/ClarionProperties.xml.bak"
        
            # Copy the backup file back to the original path
            Copy-Item -Path $backupPath -Destination $clarionPropertiesPath -Force
```
This step restores the original ClarionProperties.xml file by copying the backup file back to its original location.
```yaml
      - name: Create Build Directory
        # Create a build directory if it doesn't exist
        run: |
          $buildDirectory = "${{ github.workspace }}/build"
          if (-not (Test-Path $buildDirectory)) {
            New-Item -ItemType Directory -Path $buildDirectory
          }
```
This step creates a build directory (build) if it doesn't already exist using PowerShell. It checks if the directory exists and, if not, creates it.
**NOTE** I have my .RED file generation applications to the "working" directory. It is probably a very good idea to do similar yourself as this makes life much easier.
```yaml
      - name: Zip Contents
        # Zip the contents of the 'working' directory
        run: |
          $sourceDirectory = "${{ github.workspace }}/working"
          $zipFileName = "Build-$(Get-Date -Format 'yyyy-MM-dd-HH-mm').zip"
          $zipPath = "${{ github.workspace }}/build/$zipFileName"
          Compress-Archive -Path $sourceDirectory -DestinationPath $zipPath
```
This step zips the contents of the working directory into a timestamped zip file in the build directory using PowerShell's Compress-Archive cmdlet.
```yaml
      - name: Copy Zip to Build Directory
        # Copy the generated zip file to the build directory
        run: |
          $sourcePath = "${{ github.workspace }}/Build-*.zip"
          $destinationDirectory = "${{ github.workspace }}/build"
          $destinationPath = Join-Path $destinationDirectory (Get-ChildItem -Path $sourcePath).Name
          Copy-Item -Path $sourcePath -Destination $destinationPath -Force
```
This step copies the generated zip file from the workspace root to the build directory. It retrieves the source path of the zip file using a wildcard pattern, determines the destination path based on the build directory, and performs the copy operation.
```yaml
      - name: Disable Git LFS
        # Uninstall Git LFS
        run: |
          git lfs uninstall --skip-repo
```
This step disables Git LFS by uninstalling it. If you are using LFS in your dev directory remove this step.
```yaml
      - name: Commit changes
        # Commit the changes made during the build process
        run: |
          git config --global user.name "GitHub Actions"
          git config --global user.email "actions@github.com"
          git add -A
          git reset clarion  # Exclude the clarion directory from the commit
          git commit -m "Update files"
```
This step configures Git user information, adds all changes to the git index, excludes the clarion directory from the commit using git reset, and commits the changes with a specific message.
```yaml
      - name: Push changes
        # Push the changes to the repository
        run: git push

```
This step pushes the committed changes to the repository.