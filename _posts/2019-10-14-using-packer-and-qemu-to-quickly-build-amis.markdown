---
layout: post
title:  "Using Packer and QEMU to Quickly Build AMIs/VM Images"
date:   2019-10-14 16:00:00 -0600
categories: packer qemu ami aws
---

[Packer](https://www.packer.io/) is a super useful tool to help you create and modify AMIs and VM snapshots. All tools and scripts to generate your images can be committed to source control, and even wired up to your CI/CD tooling to allow you to fully trust your images (a post about this is coming soon).

At Pathmatics, we leverage AWS AMI's to launch fleets of EC2 instances, and in the past, we've tried to keep a manual record of how to build a base iamge for all our software. Frankly, this was error prone and didn't scale. We needed a way to have an enforcable, consistent way of building an image, and Packer was the answer. There are many great tutorials out there on how to build an AMI with Packer (starting with [Packer's](https://www.packer.io) own documentation), but one of our frustrations with using the AMI builder was the slow iteration time. It was pretty irritating to discover a syntax error in a Bash or Powershell script af the tail end of a long AMI build process.

Our solution to this was to leverage a different type of builder that Packer also supports: [QEMU](https://www.qemu.org/). QEMU is an emulator that can run on top of a bunch of different virtualization engines (in our case Hyper-V). Using the QEMU builder, we were able to snapshot the VM at any point in the provision script on our local dev machines and take the pain out of making a mistake in our provisioner scripts.

We found it useful to wrap our calls to the Packer binary in a script so we could parameterize what was the base snapshot. This allowed us to get our image to a good point, snapshot it, then add additional code. The only catch was that your provision script needs to be idempotent. But this saved us a ton of time where our image would be installing apt packages over and over again. Here is our script:

```powershell
param(
[Parameter(HelpMessage = "Action to take", Mandatory)]
[string]$Action,
[Parameter(HelpMessage = "Name of the Packer variable file")]
[string]$VariableFile = "variables.json",
[Parameter(HelpMessage = "Name of the Packer ubuntu minimmal file")]
[string]$UbuntuMinimalFile = "ubuntu-minimal.json",
[Parameter(HelpMessage = "Name of the Packer crawler file")]
[string]$CrawlerFile = "crawler.json",
[Parameter(HelpMessage = "SSH keypair file")]
[string]$SshKeyPairFile = "",
[Parameter(HelpMessage = "Image file")]
[string]$ImageFile = '',
[Parameter(HelpMessage = "Crawler output filename")]
[string]$OutputFolder = ""
)

$VariableJson = Get-Content $VariableFile | ConvertFrom-Json
$CrawlerJson = Get-Content $CrawlerFile | ConvertFrom-Json

switch ($Action)
{
    "build-ubuntu-minimal"
    {
        Remove-Item -LiteralPath "$($VariableJson.ubuntu_minimal_output_directory)" -Force -Recurse -ErrorAction Ignore
        packer build -var-file $VariableFile $UbuntuMinimalFile
    }
    "run-ubuntu-minimal"
    {
        qemu-system-x86_64.exe -boot c `
            -device "virtio-net,netdev=user.0" `
            -drive "file=$($VariableJson.ubuntu_minimal_output_directory)\\$($VariableJson.ubuntu_minimal_vm_name),if=virtio,cache=writeback,discard=ignore,format=qcow2" `
            -display sdl `
            -machine "type=pc,accel=whpx" `
            -netdev "user,id=user.0,hostfwd=tcp::3406-:22" `
            -vnc "127.0.0.1:8" `
            -m 2048 `
            -smp cpus=2 `
            -name "$($VariableJson.ubuntu_minimal_vm_name)"
    }
    "build-crawler-local"
    {   
        $AdditionalArgs = [System.Collections.ArrayList]@()
        If ($ImageFile) {
            $AdditionalArgs.Add("-var iso_url=$ImageFile") > $null
        }
        else {
            $ImageFile = ".\$($VariableJson.ubuntu_minimal_output_directory)\$($VariableJson.ubuntu_minimal_vm_name)"
        }

        If ($OutputFolder) {
            $AdditionalArgs.Add("-var crawler_output_directory=$OutputFolder") > $null
        }
        else {
            $OutputFolder = "$($CrawlerJson.variables.crawler_output_directory)"
        }
        Write-Host "Removing old local image (if it exists)"
        Remove-Item -LiteralPath "$OutputFolder" -Force -Recurse -ErrorAction Ignore
        Write-Host "Hashing Ubuntu Minimal image"
        $hash = (Get-FileHash $ImageFile -Algorithm SHA1).Hash
        Write-Host "Launching packer!"
        Invoke-Expression "packer.exe build -var-file $VariableFile -var iso_checksum=$hash $AdditionalArgs -only qemu $CrawlerFile"
    }
    "build-crawler-ec2"
    {        
        If (-Not $SshKeyPairFile) {
            Throw "You must specify a keypair file"
        }

        Write-Host "Launching packer!"
        packer build -var-file $VariableFile -only amazon-ebs -var ssh_private_key_file=$SshKeyPairFile -var ssh_keypair_name="$((Get-Item $SshKeyPairFile).Basename)" $CrawlerFile
    }
    "run-crawler-local"
    {
        If (-Not $ImageFile) {
            $ImageFile = "$($CrawlerJson.variables.crawler_output_directory)\\$($CrawlerJson.variables.crawler_vm_name)"
        }
        Write-Host $ImageFile
        qemu-system-x86_64.exe -boot c `
            -device "virtio-net,netdev=user.0" `
            -drive "file=$ImageFile,if=virtio,cache=writeback,discard=ignore,format=qcow2" `
            -display sdl `
            -machine "type=pc,accel=whpx" `
            -netdev "user,id=user.0,hostfwd=tcp::3406-:22" `
            -vnc "127.0.0.1:8" `
            -m 2048 `
            -smp cpus=2 `
            -name "$($CrawlerJson.variables.crawler_vm_name)"
    }    
    default
    {
        Write-Host "ERROR: Unknown action: $Action" -ForegroundColor Red
        Exit 1
    }
}
```

The thing to note about this script is that it builds a base image (build-ubuntu-local), and then you can iterate on the build-crawler-local step as it takes an input base image and allows you to customize the output.

One final useful tool is an EC2 instance metadata mocking tool, if you need your instance to access any AWS resources. While we ended up rolling our own, there are many [available](https://github.com/nytimes/mock-ec2-metadata) on GitHub.

Using this workflow has allowed us to rapidly iterate on legacy systems that require the custom stamping of an image/AMI.