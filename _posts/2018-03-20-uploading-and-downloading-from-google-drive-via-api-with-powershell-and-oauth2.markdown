---
layout: post
title: "Uploading and Downloading from Google Drive via API with PowerShell and OAuth 2"
date: 2018-03-20 11:39:00 -700
categories: PowerShell
tags:
- PowerShell
- Google Drive
- API
comments: true
---
<!--more-->
My organization is currently in the process of migrating from Office 365 to G Suite, as well as migrating to Team Drives from a traditional SMB file server. This has resulted in a need for an easy method to interact with Google Drive using PowerShell, as I have a number of scripts that store and read data on SMB file shares. Thanks to Montel Edwards for getting me started
with <a href="https://monteledwards.com/2017/03/05/powershell-oauth-downloadinguploading-to-google-drive-via-drive-api/">this</a> post.

I am working on a GDrive module that uses concepts from this post to mirror many built-in
PowerShell functions, such as: Get-Item, Get-ChildItem, New-Item, etc. The GDrive module would include functions such as: Get-GDriveItem, Get-GDriveChildItem, New-GDriveItem, etc.

## Create Project and OAuth 2 Tokens

Write stuff

## Get Refresh Token

Write stuff

## Download from Drive

Write stuff

### Download/Export Based on MIME Type

Write stuff

## Upload to Drive

Get the source file, encode in Base64 and get the MIME type.

{% highlight powershell %}
# Change this to the file you want to upload
$SourceFile = 'C:\Path\To\File'

# Get the source file contents and details, encode in base64
$sourceItem = Get-Item $sourceFile
$sourceBase64 = [Convert]::ToBase64String([IO.File]::ReadAllBytes($sourceItem.FullName))
$sourceMime = [System.Web.MimeMapping]::GetMimeMapping($sourceItem.FullName)
{% endhighlight %}

Set the file metadata. Additional properties can be added as needed. Reference the <a href="https://developers.google.com/drive/v3/reference/files/create#request-body">files/create</a>
documentation for additional properties.

{% highlight powershell %}

# Set the file metadata
$uploadMetadata = @{
    originalFilename = $sourceItem.Name
    name = $sourceItem.Name
    description = $sourceItem.VersionInfo.FileDescription
}
{% endhighlight %}

Build the multipart upload body from the base64 data and the file metadata.

{% highlight powershell %}
# Set the upload body
$uploadBody = @"
--boundary
Content-Type: application/json; charset=UTF-8

$($uploadMetadata | ConvertTo-Json)

--boundary
Content-Transfer-Encoding: base64
Content-Type: $sourceMime

$sourceBase64
--boundary--
"@

{% endhighlight %}

Set the upload headers and perform the upload.

{% highlight powershell %}
# Set the upload headers
$uploadHeaders = @{
    "Authorization" = "Bearer $accessToken"
    "Content-Type" = 'multipart/related; boundary=boundary'
    "Content-Length" = $uploadBody.Length
}
# Perform the upload
$response = Invoke-RestMethod -Uri 'https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart' -Method Post -Headers $uploadHeaders -Body $uploadBody
{% endhighlight %}

### Uploading to a Specific Folder

To upload to a specific folder rather than to the root of your Google Drive, you need to specify a parent ID in the file metadata.
The API documentation says that parents should be an array, but it seems to error out if you provide more than one value. I don't
fully understand the use case for multiple parents, but for simple uploads having 1 works just fine.  

The folder ID can be seen in your address bar when browsing Google Drive. For example: `https://drive.google.com/drive/folders/1hUL97Xd6tEiR-44fV7PYQ9BZaulA3ASg`

{% highlight powershell %}
$uploadMetadata = @{
    originalFilename = $sourceItem.Name
    name = $sourceItem.Name
    description = $sourceItem.VersionInfo.FileDescription
    parents = @(‘folderId’)
}
{% endhighlight %}

### Uploading to a Team Drive

If you would like to upload to a Team Drive rather than to your 'My Drive', you have to modify the $uploadMetadata and add `&supportsTeamDrives=true` to the upload URI. Similar to the 'Uploading to a Specific Folder'
scenario, we must provide a parent ID. In this case the parent ID is either the team drive ID (to upload to the root of the team drive), or a folder ID (to upload to a folder within the team drive). We also have to
provide a teamDriveId, which can be found in your address bar, just like the folder ID.

{% highlight powershell %}
$uploadMetadata = @{
    originalFilename = $sourceItem.Name
    name = $sourceItem.Name
    description = $sourceItem.VersionInfo.FileDescription
    parents = @(‘teamDriveId or folderId’)
    teamDriveId = ‘teamDriveId’
}
{% endhighlight %}

When performing the upload, we change the -Uri parameter to include the parameter `&supportsTeamDrives=true`.

{% highlight powershell %}
$response = Invoke-RestMethod -Uri 'https://www.googleapis.com/upload/drive/v3/files?uploadType=multipart&supportsTeamDrives=true' -Method Post -Headers $uploadHeaders -Body $uploadBody
{% endhighlight %}
