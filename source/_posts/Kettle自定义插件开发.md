---
title: kettle自定义插件开发
date: 2017-02-22 12:00:00
categories: 
- 大数据
tags:
- 大数据
---

## 背景

随着业务种类越来越多，kettle内置的插件也逐渐满足不了各种业务需求，因此需要通过kettle自定义插件的方式来满足业务的需求。

## kettle插件

开发kettle的业务插件也有一段时间，整理了一下大概也有十多个，大概如下：

1. ESDB Output组件：Pipeline ESDB Output组件是一个Kettle的扩展插件，用于将数据输出到ESDB（自研发的地球科学数据库）中。

1. ESDB 字段映射组件：要用于将数据字段名称映射成规范的编码。比如我们提供了一份《气象行业字段标准编码》的文件，里面包含了CIMISS字段、IDEA字段与标准编码的映射。那么在采集CIMISS数据，或者IDEA数据时，就可以通过该组件，将字段名转换为标准的编码，最后才通过ESDB Output插件，进行入库。
每个行业（如气象、水利等），都可以整理一份字段标准编码CSV文件, 然后利用该组件进行自动匹配。规范字段编码的意义，在于对于不同的系统（比如不同省份的相似系统），数据查询的时候，可以通过统一的字段进行查询。比如地表温度，可以通过gst字段进行查询，无论该数据是来源于CIMISS，还是来源IDEA，还是来源于本地客户提供的数据。

1. Pipeline 通用组件：pipeline 通用组件,为用户开发自定义转换的插件提供统一的界面展示和配置操作。

1. Pipeline 通用组件-经纬度转换组件：pipeline 通用组件 经纬度转换组件,为用户提供经纬度转换坐标系（WGS84、GCJ_02、BD_09之间相互转换）的功能。

1. Pipeline 通用组件-站点插值组件：pipeline 通用组件-插值组件,为用户提供站点插值成格点数据的功能。

1. Pipeline 通用组件-idea（intgetdata2d接口）格点解析组件：格点解析组件,为用户提供解析intgetdata2d接口格点数据的功能
1. Pipeline 列合并组件：为用户提供针对某个字段，多行数据合并成一行数据的功能，合并后的对象是一个List对象。
1. Pipeline 多值映射组件：为用户提供值映射的功能。
1. Pipeline 通用组件-idea-byte转格点：dea（qpe、qpf接口）byte转格点解析组件,为用户提供byte转格点数据的功能。由于qpe与qpf接口返回格点数据经过base64加密，因此该插件会先base64解密然后将byte转换成格点数据
1. Pipeline 通用组件-格点双线性插值：为用户格点插值的功能（双线性插值）。
1. Pipeline 通用组件-气象文件读取插件（网格文件）：
1. Pipeline 通用组件-RestClientNew插件：提供读取rest接口的功能，主要是增加超时设置，kettle自带的restClient插值没有读取超时机制，如果接口卡住，连接会一直保持，所以新增本插件支持读取超时设置功能。
1. Pipeline 通用组件-多行数据转JSON插件：多行数据转JSON插件,提供把一行或多行数据根据分组字段合并成json字符串。
1. Pipeline 图片压缩插件：用于将图片进一步压缩的组件。
1. Pipeline 图片生成gif动图插件：能够将多张图片生成多张含有时间信息的图片，并将生成后的图片聚合成一张gif动态图片。

## 开发自定义插件

本章简单记录我认为开发属于自己步骤插件的要点，要想完整搭建参考[自定义步骤插件开发](https://help.pentaho.com/Documentation/8.3/Developer_center/Create_step_plugins)。

### 四个接口

开发前提条件首先要了解四个接口


接口 | 基础类 | 主要职责
---|---|---
StepMetaInterface | BaseStepMeta | 1.存储step设置信息</br>2.验证step设置信息</br>3.序列化step设置信息</br>4.提供获取step类的方法
StepDialogInterface |  org.pentaho.di.ui.trans.step.BaseStepDialog	| step属性信息配置窗口
StepInterface | BaseStep | 执行数据行的功能流程
StepDataInterface | BaseStepData | 存储处理中的数据状态

以下我结合代码尽量

### 实现StepMetaInterface

该接口主要是对步骤元数据信息进行操作。

接口常用方法介绍：

- setDefault()：每次创建新步骤并将该步骤配置分配或设置为合理的默认值时，都会调用此方法。创建新步骤时，PDI客户端（Spoon）将使用此处设置的值。这是确保将步设置初始化为非空值的好地方。在序列化和对话框填充中，空值的处理可能很麻烦，因此大多数PDI步骤实现对于所有步骤设置都坚持非空值。
- clone()：在PDI客户端中复制步骤时，将调用此方法。它返回步骤元对象的深层副本。如果步骤配置存储在可修改的对象（例如列表或自定义帮助对象）中，则实现类必须创建正确的深层副本，这一点至关重要。

- getXML()：每当步骤将其设置序列化为XML时，PDI都会调用此方法。在PDI客户端中保存转换时会调用它。该方法返回一个XML字符串，其中包含序列化的步骤设置。该字符串包含一系列XML标记，每个设置一个标记。辅助类org.pentaho.di.core.xml.XMLHandler构造XML字符串。
- loadXML()：每当步骤从XML读取其设置时，PDI都会调用此方法。包含步骤设置的XML节点作为参数传入。再次，帮助程序类 org.pentaho.di.core.xml.XMLHandler从XML节点读取步骤设置。
- saveRep()：每当步骤将其设置保存到PDI存储库时，PDI都会调用此方法。作为第一个参数传入的存储库对象提供了一组用于序列化步骤设置的方法。调用存储库序列化方法时，该步骤将传入的转换ID和步骤ID用作标识符。
- readRep()：每当步骤从PDI存储库中读取其配置时，PDI都会调用此方法。使用存储库序列化方法时，参数中给出的步骤ID用作标识符。

获取其他实例
- public StepDialogInterface getDialog()
- public StepInterface getStep()
- public StepDataInterface getStepData()


StepMetaInterface必须使用Step Java注释对实现的类进行注释。提供以下注释属性：
属性|描述
---|---
id | 该步骤的全局唯一ID
image | 步骤的png图标图像的资源位置
name | 该步骤的简短标签
description	| 该步骤的详细说明
categoryDescription |	步骤的类别应显示在PDI步骤树中。例如输入，输出，变换等。
i18nPackageName |如果i18nPackageName在批注属性中提供了该属性，则将name，description和categoryDe​​scription的值解释为 i18n相对于给定包中包含的消息束的键。</br>可以以扩展形式的i18n:<packagename>键来提供键，以指定与i18nPackageName属性中给定的包不同的包。

### 实现StepDialogInterface

该接口主要用于实现窗口的功能，以及设置属性信息的入口。打开步骤设置时候都会实例化dialog传入到StepMetaInterface接口对象并调用open()方法。

接口常用方法介绍：

- open()：仅在确认或取消对话框后，此方法才返回。

### 实现StepInterface

StepInterface当转换运行时，类实现负责实际的行处理。

![StepInterface](https://note.youdao.com/yws/public/resource/9ad28471772957ddecb1e977ed1a2ac0/xmlnote/9B1127DC0D95467C84BDA85446348950/12098)

接口常用方法介绍：

- init()：当转换准备开始执行时，将调用该方法初始化资源。

- processRow()：转换开始执行该方法，读取上一步骤传递下来的行数据，直到没有行就调用setOutputDone()返回false。

- dispose()：转换完成后，PDI将调用该方法取消init()分配的资源。

### 实现StepDataInterface

类实现StepInterface不会在其任何字段中存储处理状态。取而代之的StepDataInterface是，使用一个附加的类实现来存储处理状态，包括状态标志，索引，缓存表，数据库连接，文件句柄等

## 部署步骤插件

1. 创建一个包含您的插件类和资源的JAR文件
1. 创建一个新文件夹，为其命名，然后将您的JAR文件放在该文件夹中
1. 将刚创建的插件文件夹放置在特定位置，以供PDI查找。根据您使用PDI的方式，您需要按如下方式将插件文件夹复制到一个或多个位置。
    - 部署到PDI客户端（Spoon）或Carte：
    - 将plugin文件夹复制到以下位置： design-tools / data-integration / plugins / steps。
    - 重新启动PDI客户端。重新启动PDI客户端后，可以使用新的作业条目。

## 参考

[Kettle开发中心](https://help.pentaho.com/Documentation/8.3/Developer_center/Embed_and_extend_PDI_functionality)

[扩展Pentaho数据集成](https://help.pentaho.com/Documentation/8.3/Developer_center/Extend_Pentaho_Data_Integration)

[自定义步骤插件开发](https://help.pentaho.com/Documentation/8.3/Developer_center/Create_step_plugins)