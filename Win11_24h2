# Define the base path for all operations
$basePath = "e:\!dism"

function Get-Wim {
    param (
        [string]$isoFolderPath = "$basePath\iso\win1124h2",
        [string]$mountPath = "$basePath\mount",
        [string]$baseWimPath = "$basePath\Windows_11_24h2_Base.wim",
        [string]$enterpriseWimPath = "$basePath\Windows_11_24h2_Enterprise_Base.wim"
    )

    # Get the first and only ISO file in the folder
    $isoPath = Get-ChildItem -Path $isoFolderPath -Filter *.iso | Sort-Object LastWriteTime -Descending | Select-Object -First 1

    if ($null -eq $isoPath) {
        Write-Output "No ISO file found in the specified folder."
        return
    }

    # Mount the ISO
    Mount-DiskImage -ImagePath $isoPath.FullName
    $diskImage = Get-DiskImage -ImagePath $isoPath.FullName

    # Create mount directory if it doesn't exist
    if (-Not (Test-Path $mountPath)) {
        New-Item -ItemType Directory -Path $mountPath
    }

    # Mount the ISO to the directory
    $mountResult = Mount-DiskImage -ImagePath $isoPath.FullName -PassThru | Get-Volume | Select-Object -ExpandProperty DriveLetter
    $mountDrive = "$($mountResult):\"

    # Copy the WIM file, overwriting if it exists
    Copy-Item -Path "$mountDrive\sources\install.wim" -Destination $baseWimPath -Force

    # Dismount the ISO
    Dismount-DiskImage -ImagePath $isoPath.FullName

    Write-Output "Base WIM file extracted to $baseWimPath"

    # Get the indexes of all images in the base WIM
    $images = Get-WindowsImage -ImagePath $baseWimPath

    # Find the index for the Enterprise image
    $enterpriseImage = $images | Where-Object { $_.ImageName -eq "Windows 11 Enterprise" }

    if ($null -eq $enterpriseImage) {
        Write-Output "Enterprise image not found in the base WIM."
        return
    }

    # Export the Enterprise image to a new WIM file
    Export-WindowsImage -SourceImagePath $baseWimPath -SourceIndex $enterpriseImage.ImageIndex -DestinationImagePath $enterpriseWimPath -DestinationName "Windows 11 Enterprise 24H2" -ScratchDirectory $mountPath -CheckIntegrity

    Write-Output "Enterprise WIM file exported to $enterpriseWimPath"
}

function Get-WimPackages {
    param (
        [string]$enterpriseWimPath = "$basePath\Windows_11_24h2_Enterprise_Base.wim",
        [string]$wimMountPath = "$basePath\mount",
        [string]$packagesOutputPath = "$basePath\win1124h2_packages_base.txt"
    )

    # Create mount directory if it doesn't exist
    if (-Not (Test-Path $wimMountPath)) {
        New-Item -ItemType Directory -Path $wimMountPath
    }

    # Mount the Enterprise WIM file
    Mount-WindowsImage -ImagePath $enterpriseWimPath -Path $wimMountPath -Index 1

    # Get all provisioned packages
    $packages = Get-AppxProvisionedPackage -Path $wimMountPath

    # Export the list of provisioned packages to a text file (DisplayName only)
    $packages | Select-Object DisplayName | Format-Table -AutoSize | Out-String | Set-Content -Path $packagesOutputPath -Force

    # Dismount the WIM
    Dismount-WindowsImage -Path $wimMountPath -Discard

    Write-Output "Provisioned packages list exported to $packagesOutputPath"
}

function New-Wim {
    param (
        [string]$enterpriseWimPath = "$basePath\Windows_11_24h2_Enterprise_Base.wim",
        [string]$testWimPath = "$basePath\Windows_11_24h2_Test.wim",
        [string]$wimMountPath = "$basePath\mount",
        [string]$scratchDir = "$basePath\scratch",
        [string]$imageName = "Windows 11 Enterprise 24H2",
        [string]$packagesOutputPath = "$basePath\win1124h2_packages_modified.txt"
    )

    # Get the indexes of all images in the Enterprise WIM
    $images = Get-WindowsImage -ImagePath $enterpriseWimPath

    # Find the index for the Enterprise image
    $enterpriseImage = $images | Where-Object { $_.ImageName -like "*Enterprise*" }

    if ($null -eq $enterpriseImage) {
        Write-Output "Enterprise image not found in the Enterprise WIM."
        return
    }

    # Mount the Enterprise image
    $wimMountPath = "$basePath\mount"
    if (-Not (Test-Path $wimMountPath)) {
        New-Item -ItemType Directory -Path $wimMountPath
    }
    Mount-WindowsImage -ImagePath $enterpriseWimPath -Path $wimMountPath -Index $enterpriseImage.ImageIndex

    # Get all provisioned packages
    $packages = Get-AppxProvisionedPackage -Path $wimMountPath

    # Create an array of whitelisted apps
    $whitelistedApps = @(
        "Microsoft.ApplicationCompatibilityEnhancements",
        "Microsoft.AV1VideoExtension",
        "Microsoft.AVCEncoderVideoExtension",
        "Microsoft.DesktopAppInstaller",
        "Microsoft.GetHelp",
        "Microsoft.HEIFImageExtension",
        "Microsoft.HEVCVideoExtension",
        "Microsoft.MPEG2VideoExtension",
        "Microsoft.RawImageExtension",
        "Microsoft.ScreenSketch",
        "Microsoft.SecHealthUI",
        "Microsoft.StorePurchaseApp",
        "Microsoft.Todos",
        "Microsoft.VP9VideoExtensions",
        "Microsoft.WebMediaExtensions",
        "Microsoft.WebpImageExtension",
        "Microsoft.Windows.Photos",
        "Microsoft.WindowsAlarms",
        "Microsoft.WindowsCalculator",
        "Microsoft.WindowsCamera",
        "Microsoft.WindowsNotepad",
        "Microsoft.WindowsSoundRecorder",
        "Microsoft.WindowsStore",
        "Microsoft.WindowsTerminal",
        "MicrosoftWindows.Client.WebExperience"
    )

    # Loop through all provisioned packages and remove the ones not in the whitelist
    foreach ($package in $packages) {
        if ($whitelistedApps -contains $package.DisplayName) {
            Write-Output "Whitelisted package: $($package.DisplayName)"
        } else {
            Write-Output "Removing package: $($package.DisplayName)"
            Remove-AppxProvisionedPackage -Path $wimMountPath -PackageName $package.PackageName -Verbose
        }
    }

    # Get all provisioned packages
    $packages = Get-AppxProvisionedPackage -Path $wimMountPath

    # Export the list of provisioned packages to a text file (DisplayName only)
    $packages | Select-Object DisplayName | Format-Table -AutoSize | Out-String | Set-Content -Path $packagesOutputPath -Force

    # Dismount, save, and check the integrity of the WIM
    Dismount-WindowsImage -Path $wimMountPath -Save -CheckIntegrity

    Write-Output "Enterprise image modified and saved."

    # Export the Enterprise image to a new WIM file
    Export-WindowsImage -SourceImagePath $enterpriseWimPath -SourceIndex $enterpriseImage.ImageIndex -DestinationImagePath $testWimPath -DestinationName $imageName -ScratchDirectory $scratchDir -CheckIntegrity

    Write-Output "Enterprise image exported to $testWimPath with name '$imageName'"
}

# Example usage
Get-Wim
#Get-WimPackages
New-Wim
