---
layout: post 
title: Eclipse中使用Jupiter插件进行代码Review
summary: Jupiter 是为Eclipse开发的一款开源code review工具插件，使用Jupiter可以在团队中进行code review，可以通过SVN等将review结果在团队内进行共享
tags: eclipse，code review
---

{{ page.title }}
================

## About Jupiter

Jupiter 是为Eclipse开发的一款开源code review工具插件，使用Jupiter可以在团队中进行code review，可以通过SVN等将review结果在团队内进行共享。并能建立review代码段与源代码直接的关系，在EclipseIDE中方便进行跳转查看。Jupiter的项目地址：https://code.google.com/p/jupiter-eclipse-plugin.

### Install

Eclipse中 "Help | Install New Software..." 通过插件地址进行安装。地址：http://jupiter-eclipse-plugin.googlecode.com/svn/trunk/site

## Review

Jupiter的Code Review流程为以下几步：

   1. 建立review任务：代码修改者建立review任务，指定需评审的代码文件、参与评审的人员、定义问题类型及严重级别等。

   2. 个人评审阶段：开始个人独自评审，将可能出现的问题加入问题列表。

   3. 团队评审阶段：团队成员坐在一起，讨论个人评审阶段的问题，确定是否需要修复并分配解决人员。

   4. 问题修复阶段：每个人修复分配给自己的问题，修复后修改问题状态。

### 创建ReviewID

   1. 在"Package Explorer"或"Navigater"里，右键点击项目名称，选择"Properties"显示项目属性窗口。
   2. 选择Review标签，点击“New”新建 Review ID，输入ID和描述。
   3. 选择本次需要Review的文件。
   4. 添加Review的人员。
   5. 选择评审负责人。
   6. 设置问题类型及严重级别和filter，根据实际情况修改或用默认值。
   7. 完成之后生成.jupiter文件，提交到SVN。

### 个人评审

   1. 从SVN上更新最新的源代码和.jupiter文件，在Eclipse里“Window”-“Open Perspective”-“Other”选择“Review”打开Review视图，选择“Individual Phase”
   2. 选择Project Name(项目名称)，Review ID（review任务）和Reviewer ID（评审人员）
   3. 选择需要Review的文件开始Review代码。
   4. 发现问题时，光标停在问题代码出右键选择“Add Review Issue......”
   5. 在“Review Editor”里选择问题类型及严重性，添加概要和详细描述，保存。可以看到增加了Review问题的代码会在行首处有标记。
   6. 个人评审完毕后将Jupiter评审数据目录(默认为review)下的数据上传到SVN。

### 团队评审

团队成员从SVN上更新最新的Review数据，从review试图中选择“Team Phase”，点击“Review Table”中的问题会跳到对应的代码，一起讨论代码是否确实存在问题，在“Review Editor”里分配修复人员及解决方式，保存。

### 问题修复

个人回更新最新的review数据，从review试图中选择“Rework Phase”,会在“Review Table”里列出分配给自己的问题，逐一修复，并在“Review Editor”将问题状态改为“Resovled”，保存并将review数据上传到SVN。


#### 参考

   1.  官方文档：https://code.google.com/p/jupiter-eclipse-plugin/wiki/UserGuide