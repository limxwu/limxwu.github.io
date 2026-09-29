---
title: 合并B站下载视频与音频脚本
id: a76e98a2-8498-4c30-ad7e-b47d34286266
date: 2023-12-13 23:08:50
auther: limxwu
cover: 
excerpt: 脚本如下 $fileList = Get-ChildItem -File$filenames=@()foreach ($filename in $fileList) {    if ($filename -match '\.mp4$'){        ffmpeg -i ".\${file
permalink: /archives/1702480129449
categories:
tags: 
---

脚本如下

```powershell
$fileList = Get-ChildItem -File
$filenames=@()
foreach ($filename in $fileList) {
    if ($filename -match '\.mp4$'){
        ffmpeg -i ".\${filename}.mp4" -i ".\${filename}.m4a" -c:v copy -c:a aac "${filename}.b.mp4"
    }
}
```

这段 PowerShell 脚本执行了以下操作：

**1. 获取当前文件夹下的文件列表：**

```powershell
$fileList = Get-ChildItem -File
```

这行代码使用 Get-ChildItem -File 获取当前文件夹下的所有文件，并将结果存储在 $fileList 变量中。

**2. 遍历文件列表，判断文件名是否以 ".mp4" 结尾：**

```powershell
$filenames = @() 
foreach ($filename in $fileList) { 
    if ($filename -match '\.mp4$') { 
        # 执行以下操作 
    } 
}
```

这部分使用 foreach 循环遍历 $$ 检查文件名是否以 ".mp4" 结尾。如果是，就执行了一个操作。

**3. 使用 FFmpeg 合并视频和音轨：**

```powershell
ffmpeg -i ".\${filename}.mp4" -i ".\${filename}.m4a" -c:v copy -c:a aac "${filename}.b.mp4"
```

如果文件名以 ".mp4" 结尾，就调用 FFmpeg 命令，合并以该文件名为基础的 ".mp4" 和 ".m4a" 文件，并输出到一个新的文件，文件名为 "${filename}" 来引用文件名变量。

总体来说，这段脚本的作用是查找当前文件夹下以 ".mp4" 结尾的文件，然后使用 FFmpeg 将这些文件对应的视频和音频文件合并，并输出到一个新的 ".b.mp4" 文件。

