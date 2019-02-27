[[_TOC_]]

# 步骤说明
- ## 创建Build Pipeline，设置代码源
&emsp;&emsp;点击创建Build的Pipeline
![image.png](/.attachments/image-071775ef-6030-40bc-955e-3d76a8751474.png)
&emsp;&emsp;选择代码仓库
![image.png](/.attachments/image-43ec1440-ce03-4184-8fbf-42f4bb0f12d4.png)
&emsp;&emsp;选择Empty job
![image.png](/.attachments/image-d971b803-5397-4e78-877d-0bb891d2f43d.png)
&emsp;&emsp;选择构件环境，这里选择Hosted VS2017
![image.png](/.attachments/image-685d3b83-87f5-4876-a068-e534c3cb341b.png)
>构件环境，即是包括所有相关需要的工具的虚拟机环境，大致都有包括nuget docker dotnet等构件环境，应选择的不同而不同，点击**Pool information**可以看到相关环境配置

&emsp;&emsp;设置Agent jon，这里也设置pool是Hosted VS2017
![image.png](/.attachments/image-efb5dfa0-f92f-42da-bab4-27c8cbcc3eab.png)





- ## 添加dotnet restore

&emsp;&emsp;添加一个dotnet restore的task
![image.png](/.attachments/image-24274c8d-2066-419a-ac96-3ecfd4bd43e5.png)
![image.png](/.attachments/image-fac0ebac-ddfc-4fe2-9832-5e4eeef30a0e.png)


- ## 创建构件输出的文件夹
&emsp;&emsp;类似上面操作，点击加好添加Powershell的task，方便创建文件夹等操作，powershell操作就当做是windows虚拟机的命令操作即可
![image.png](/.attachments/image-87a6970b-a6f2-4b2c-ab2a-3699578f5d71.png)
这里在构件主目录下创建了**OutPutBuilds**的文件夹


- ## 添加dotnet build 
&emsp;&emsp;类似上面操作。点击添加按钮，添加dotnet build的task
![image.png](/.attachments/image-3b141424-82a5-40d3-986d-f0764a03e14e.png)
![image.png](/.attachments/image-5a550aca-4162-4c31-8617-541d5b9aefe9.png)

>这里说明下，目录路径：工作区默认相对路径是和代码仓库路径相同，因为它的原理是从代码仓库拉取到本地虚拟机的根目录中。
工作区根目录可以打印出来：大概在 **“D:\a\1\s\”**
发布区根目录在：**”D:\a\1\a\”**  根据名称不同而不同
变量```$(Build.ArtifactStagingDirectory)``` **发布区**
变量```$(System.DefaultWorkingDirectory)``` **工作区**
通过下面增加一个powershell可以打印出来，其实任何问题都可以通过powershell命令打印
![image.png](/.attachments/image-6ab5eaf3-0100-45ae-9e7a-61d40f629e89.png)

- ## 添加dotnet push
&emsp;&emsp;类似上面操作，点击添加按钮，增加发布到nuget的命令
![image.png](/.attachments/image-4e1f739a-3233-47aa-8145-fb2cbae44a2d.png)

# YAML结果

```yaml
resources:
- repo: self
queue:
  name: Hosted VS2017
steps:
- task: DotNetCoreCLI@2
  displayName: 'dotnet restore'
  inputs:
    command: restore

    projects: quarrierTestPackage/quarrierTestPackage/quarrierTestPackage.csproj

    vstsFeed: '860684f5-3856-48f5-8b58-78bf58229472'


- powershell: |
   echo "-----------------create output files--------------------------------"
   mkdir $(Build.ArtifactStagingDirectory)\OutPutBuilds
  displayName: 'PowerShell Script'

- task: DotNetCoreCLI@2
  displayName: 'dotnet build'
  inputs:
    projects: quarrierTestPackage/quarrierTestPackage/quarrierTestPackage.csproj

    arguments: '--output $(Build.ArtifactStagingDirectory)\OutPutBuilds'


- powershell: |
   echo "--------------------------echo files------------------------------------"
   echo $(Build.ArtifactStagingDirectory)\OutPutBuilds
   #echo $(Build.ArtifactStagingDirectory)
   #echo $(System.DefaultWorkingDirectory)
   dir $(Build.ArtifactStagingDirectory)\OutPutBuilds
  displayName: 'PowerShell Script'

- task: DotNetCoreCLI@2
  displayName: 'dotnet push'
  inputs:
    command: push

    packagesToPush: '$(Build.ArtifactStagingDirectory)\OutPutBuilds/*.nupkg'

    publishVstsFeed: 'cf1bf3a7-eb1a-46a1-af1b-98491f96c64d'
```