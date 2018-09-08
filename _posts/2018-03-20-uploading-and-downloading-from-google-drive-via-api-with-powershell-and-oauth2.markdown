---
layout: post
title: "Uploading and Downloading from Google Drive via API with PowerShell and OAuth 2"
date: 2018-03-20 11:39:00 -700
modified: 2018-09-03 22:43 -700
categories: PowerShell
tags:
    - PowerShell
    - Automation
    - Google Drive
    - API
comments: true
redirect_from:
    - /powershell/2018/03/20/uploading-and-downloading-from-google-drive-via-api-with-powershell-and-oauth2.html
---

My organization is currently in the process of migrating from Office 365 to G Suite, as well as migrating to Team Drives from a traditional SMB file server. This has resulted in a need for an easy method to interact with Google Drive using PowerShell, as I have a number of scripts that store and read data on SMB file shares.<!--more--> Thanks to Montel Edwards for getting me started with <a href="https://monteledwards.com/2017/03/05/powershell-oauth-downloadinguploading-to-google-drive-via-drive-api/" target="_blank">his post</a>.

I also highly recommend checking out the <a href="https://github.com/scrthq/PSGSuite" target="_blank">PSGsuite</a> module from Nate Ferrell. It may already accomplish what you're looking for. And if not, contribute to it!

## Create Project and OAuth 2 Tokens

The article from Montel Edwards linked above has instructions for this piece. The README for <a href="https://github.com/MVKozlov/GMGoogleDrive" target="_blank">GMGoogleDrive</a> also has some great instructions. You'll need to have a project with access to the Google Drive API, and a refresh token, client ID, and client secret.

## Get Access Token

{% highlight powershell %}
# Set the Google Auth parameters. Fill in your RefreshToken, ClientID, and ClientSecret
$params = @{
    Uri = 'https://accounts.google.com/o/oauth2/token'
    Body = @(
        "refresh_token=$RefreshToken", # Replace $RefreshToken with your refresh token
        "client_id=$ClientID",         # Replace $ClientID with your client ID
        "client_secret=$ClientSecret", # Replace $ClientSecret with your client secret
        "grant_type=refresh_token"
    ) -join '&'
    Method = 'Post'
    ContentType = 'application/x-www-form-urlencoded'
}
$accessToken = (Invoke-RestMethod @params).access_token
{% endhighlight %}

## Download from Drive

Downloading from Drive can be done two different ways:
1. Exporting Google Apps formats to standard formats.
2. Downloading binary data as-is.

We can pull the file metadata and then look at the MIME type to determine which method to use.

First we have to provide the fileId, the destinationPath, and set the authentication headers.

{% highlight powershell %}
# Set the authentication headers
$headers = @{
    'Authorization' = "Bearer $accessToken"
    'Content-type' = 'application/json'
}

# Change this to the id of the file you want to download. Right click a file and click 'Get shareable link' to find the ID
$fileId = 'fileId'

# Change this to where you want the resulting file to be placed. Just the path; do not include the file name
$destinationPath = 'Path\To\Store\File'
{% endhighlight %}

We can now use this data to get the file metadata, determine which format to download the file in, and then download or export the file.

This will download the file to `$destinationPath` with the name of the file in Google Drive.

{% highlight powershell %}
# Get the file metadata
$fileMetadata = Invoke-RestMethod -Uri "https://www.googleapis.com/drive/v3/files/$fileId" -Headers $headers

# If this is a Google Apps filetype, export it
if ($fileMetadata.mimetype -like 'application/vnd.google-apps.*') {
    # Determine which mimeType to use when exporting the files
    switch ($fileMetadata.mimetype) {
        'application/vnd.google-apps.document' {
            $exportMime = 'application/vnd.openxmlformats-officedocument.wordprocessingml.document'
            $exportExt = '.docx'
        }
        'application/vnd.google-apps.presentation' {
            $exportMime = 'application/vnd.openxmlformats-officedocument.presentationml.presentation'
            $exportExt = '.pptx'
        }
        'application/vnd.google-apps.spreadsheet' {
            $exportMime = 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet'
            $exportExt = '.xlsx'
        }
        'application/vnd.google-apps.drawings' {
            $exportMime = 'image/png'
            $exportExt = '.png'
        }
        'application/vnd.google-apps.script' {
            $exportMime = 'application/vnd.google-apps.script+json'
            $exportExt = '.json'
        }
    }
    $exportFileName = "$destinationPath\$($fileMetadata.name)$exportExt"
    Invoke-RestMethod -Uri "https://www.googleapis.com/drive/v3/files/$fileId/export?mimeType=$exportMime" -Method Get -OutFile $exportFileName -Headers $headers
} else {
    # If this is a binary file, just download it as-is.
    $exportFileName = "$destinationPath\$($fileMetadata.name)"
    Invoke-RestMethod -Uri "https://www.googleapis.com/drive/v3/files/${fileId}?alt=media" -Method Get -OutFile $exportFileName -Headers $headers
}
{% endhighlight %}

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

## Completed Scripts

These could very easily be converted to functions, but the intent was to provide a framework on which to build, not a complete solution.

### Download

{% gist 43597a3f07b990a4a48a8ef852fb5d8c %}

### Upload

{% gist dc804357bb10ff7522d0e21ddfdf9398 %}
