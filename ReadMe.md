```yaml
name: Clarion Build
name: Clarion Build

on:
  push:
    branches:
      # Branches that this action will run, probably not a good idea on master :)
      - master

jobs:
  build:
    runs-on: windows-latest

    steps:
      - name: Install Git LFS
        # Install Git LFS and display version information
        run: |
          git lfs install --skip-repo
          git lfs version

      - name: Checkout code
        # Checkout the repository's code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0
          lfs: true

      - name: Pull from Private Repository
        # Pull code from a private repository into the 'clarion' directory
        uses: actions/checkout@v3
        with:
          repository: msarson/Clarion11-DevMachine
          path: clarion
          ref: master
          token: ${{ secrets.MARK_SECRET }}
          lfs: true

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

      - name: Setup .NET Framework 4.6
        # Install .NET Framework 4.6 using Chocolatey
        run: |
          choco install dotnet4.6 --ignore-checksums -y
          refreshenv

      - name: Setup MSBuild
        # Setup MSBuild for building the project
        uses: microsoft/setup-msbuild@v1.1

      - name: Generate txa files, they will be named UPS
        # Generate txa files using ClaInterface.exe
        run: |
          $claInterfacePath = "${{ github.workspace }}\clarion\clainterface\ClaInterface.exe"
          $arguments = "COMMAND=BUILDAPP INPUT=${{ github.workspace }}\vcTestWorkflow  OUTPUT=${{ github.workspace }}"
          # Start the process in the background
          Start-Process -FilePath $claInterfacePath -ArgumentList $arguments -NoNewWindow -PassThru
          # Wait for the process to complete
          Wait-Process -Name ClaInterface

      - name: Import txa files, only way to do this is list them
        # Import txa files into the project
        run: |
          ${{ github.workspace }}/clarion/bin/clarioncl.exe  /ConfigDir "${{ github.workspace }}/appdata" /up_createappVC ${{ github.workspace }}\Helloworld.sln ${{ github.workspace }}\data.ups data
          ${{ github.workspace }}/clarion/bin/clarioncl.exe  /ConfigDir "${{ github.workspace }}/appdata" /up_createappVC ${{ github.workspace }}\Helloworld.sln ${{ github.workspace }}\HelloWorld.ups HelloWorld

      - name: Generate Applications
        # Generate applications using Clarion's clarioncl.exe
        run: |
          ${{ github.workspace }}/clarion/bin/clarioncl.exe  /ConfigDir "${{ github.workspace }}\appdata" /ag "${{ github.workspace }}\Helloworld.sln"

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

      - name: Restore ClarionProperties.xml
        # Restore the original ClarionProperties.xml file
        run: |
            $clarionPropertiesPath = "${{ github.workspace }}/appdata/ClarionProperties.xml"
            $backupPath = "${{ github.workspace }}/appdata/ClarionProperties.xml.bak"
        
            # Copy the backup file back to the original path
            Copy-Item -Path $backupPath -Destination $clarionPropertiesPath -Force

      - name: Create Build Directory
        # Create a build directory if it doesn't exist
        run: |
          $buildDirectory = "${{ github.workspace }}/build"
          if (-not (Test-Path $buildDirectory)) {
            New-Item -ItemType Directory -Path $buildDirectory
          }

      - name: Zip Contents
        # Zip the contents of the 'working' directory
        run: |
          $sourceDirectory = "${{ github.workspace }}/working"
          $zipFileName = "Build-$(Get-Date -Format 'yyyy-MM-dd-HH-mm').zip"
          $zipPath = "${{ github.workspace }}/build/$zipFileName"
          Compress-Archive -Path $sourceDirectory -DestinationPath $zipPath
      
      - name: Copy Zip to Build Directory
        # Copy the generated zip file to the build directory
        run: |
          $sourcePath = "${{ github.workspace }}/Build-*.zip"
          $destinationDirectory = "${{ github.workspace }}/build"
          $destinationPath = Join-Path $destinationDirectory (Get-ChildItem -Path $sourcePath).Name
          Copy-Item -Path $sourcePath -Destination $destinationPath -Force

      - name: Disable Git LFS
        # Uninstall Git LFS
        run: |
          git lfs uninstall --skip-repo

      - name: Commit changes
        # Commit the changes made during the build process
        run: |
          git config --global user.name "GitHub Actions"
          git config --global user.email "actions@github.com"
          git add -A
          git reset clarion  # Exclude the clarion directory from the commit
          git commit -m "Update files"

      - name: Push changes
        # Push the changes to the repository
        run: git push

```