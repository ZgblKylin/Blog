# Windows上配置OpenSSH服务器和VSC远程开发

## 安装OpenSSH Server

详见[Get started with OpenSSH for Windows | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Get-Service -Name sshd | Set-Service -StartupType Automatic
Start-Service sshd
```

## 配置密钥登录

详见[Key-based authentication in OpenSSH for Windows | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)。

客户端生成密钥：

```powershell
ssh-keygen -t ecdsa
```

会在`C:\Users\username\.ssh`下生成`id_ecdsa`私钥和`id_ecdsa.pub`公钥文件。

* 若服务端账号为普通用户（无UAC权限），则将`id_ecda.pub`内容复制到服务端`C:\Users\用户名\.ssh\authorized_keys`文件中：
  
  ```powershell
  # Get the public key file generated previously on your client
  $authorizedKey = Get-Content -Path $env:USERPROFILE\.ssh\id_ecdsa.pub
  
  # Generate the PowerShell to be run remote that will copy the public key file generated previously on your client to the authorized_keys file on your server
  $remotePowershell = "powershell New-Item -Force -ItemType Directory -Path $env:USERPROFILE\.ssh; Add-Content -Force -Path $env:USERPROFILE\.ssh\authorized_keys -Value '$authorizedKey'"
  
  # Connect to your server and run the PowerShell using the $remotePowerShell variable
  ssh username@domain1@contoso.com $remotePowershell
  ```

* 若服务端账号为管理员账号（有UAC权限），则需要以管理员权限将公钥填充至`C:\ProgramData\ssh\administrators_authorized_keys`，并确保该文件权限为仅管理员可读写：
  
  ```powershell
  # Get the public key file generated previously on your client
  $authorizedKey = Get-Content -Path $env:USERPROFILE\.ssh\id_ecdsa.pub
  
  # Generate the PowerShell to be run remote that will copy the public key file generated previously on your client to the authorized_keys file on your server
  $remotePowershell = "powershell Add-Content -Force -Path $env:ProgramData\ssh\administrators_authorized_keys -Value '''$authorizedKey''';icacls.exe ""$env:ProgramData\ssh\administrators_authorized_keys"" /inheritance:r /grant ""Administrators:F"" /grant ""SYSTEM:F"""
  
  # Connect to your server and run the PowerShell using the $remotePowerShell variable
  ssh username@domain1@contoso.com $remotePowershell
  ```

* 关闭服务端密码登录：管理员权限修改`C:\ProgramData\ssh\sshd_config`，取消`# PasswordAuthentication yes`一行注释，改为`PasswordAuthentication no`，保存；
* 重启`sshd`服务。

客户端可在`C:\Users\username\.ssh\config`中配置服务端信息，降低指令复杂度：

```properties
Host 主机代号
  HostName
  主机ip/域名
  User 主机账号
  PreferredAuthentications
  publickey IdentityFile id_ecdsa私钥路径
```

后续即可直接通过`ssh 主机代号` 连接。

## 管理员账号权限限制

Windows的OpenSSH Server，在使用管理员账号登录时，SSH会话并非如Linux上那样默认普通权限，而是默认已经提权至SYSTEM权限。根据[GItHub Issue](https://github.com/PowerShell/Win32-OpenSSH/issues/1652#issuecomment-685865235)，该现象为设计如此。

若要限制管理员权限，则有两个办法：

* 使用非管理员用户登录SSH；
* 根据issue末尾回复：
  * 安装busybox-win32：`winget install --id frippery.busybox-w32`；
  * 根据[Release Notes: busybox-w32 FRP-5007-g82accfc19 (2023-05-28)](https://frippery.org/busybox/release-notes/FRP-5007.html)，制作一个名为`pdrop.exe`的符号链接指向该程序：cmd下运行`mklink pdrop.exe C:\Users\用户名\AppData\Local\Microsoft\WinGet\Links\busybox.exe`，通过该链接会启动降权至普通权限的`powershell`；
  * 根据[OpenSSH Server configuration for Windows | Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh-server-configuration#configuring-the-default-shell-for-openssh-in-windows)，将sshd默认会话入口指向`pdrop.exe`：PowerShell管理员权限运行`New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value ".....\pdrop.exe" -PropertyType String -Force`

## VSCode Remote SSH连接问题

### gcim访问拒绝

若使用busybox降权，或非管理员账户，则VSCode Remote SSH会报错`Get-CimInstance : Access denied`。

根据[GitHub Issue](https://github.com/microsoft/vscode-remote-release/issues/2648#issuecomment-1646047396)回复，可通过将账户添加至WMI权限管理中来规避该问题：

* 在服务端，以管理员账号启动`compmgmt.msc`；
* 进入`服务和应用程序`——`WMI控件`——右键`属性`——`安全`选项卡——`安全设置`按钮——`高级`按钮——`添加`按钮——`选择主体`——`输入账号名`——`检查名称`——`确定`——勾选所有权限——一路`确定`关闭所有对话框；
* 重启`winmgmt`服务（`Windows Management Instrmentation`）和`sshd`服务；

配置完成后，可通过`ssh`连接至主机，通过`busybox id`确认并非SYSTEM会话，然后尝试执行`gcim win32_process`，若能正确打印进程列表，则配置成功。

### PSReadLineHistoryFile报错

降权后的SSH会话中，有可能出现PSReadLineHistoryFile报错。

根据[After demotion: Access to the path &#39;PSReadLineHistoryFile_-$NUMBER&#39; is denied · Issue #2403 · PowerShell/PSReadLine · GitHub](https://github.com/PowerShell/PSReadLine/issues/2403)，升级对应模块：`Install-Module -Name PSReadLine`；

### PowerShell中文乱码

某些情况下，VSCode Remote SSH接连过程中，会出现PowerShell中文乱码的现象。

具体出现原因未知，我的两台电脑中只有一台出现此现象，该现象当且仅当使用了`busybox`作为`sshd`会话入口时才会出现，复现方式是：

* `ssh 主机`进入busybox会话时，中文正常显示；
* `ssh 主机 powershell`（VSCode Remote SSH使用此方式连接）直接进入PowerShell会话时，中文乱码，包括欢迎文字和后续中文内容；

此时，可以通过将PowerShell默认编码设置为UTF-8解决，缺点是通过第二种方式连接时，PowerShell会话默认使用英文而非中文：

* 在主机PowerShell会话中，执行`notepad $PROFILE`；
* 添加代码行`$OutputEncoding = [console]::InputEncoding = [console]::OutputEncoding = New-Object System.Text.UTF8Encoding`；
