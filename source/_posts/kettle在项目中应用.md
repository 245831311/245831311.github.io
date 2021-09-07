---
title: kettle在项目中应用
date: 2017-02-22 12:00:00
categories: 
- 大数据
tags:
- 大数据
---

## 背景

随着发展，公司后续对接的项目对大数据量的存储和清洗需求越来越旺盛，我有幸接触这个项目并成为其中开发一员，致力于构建了一个底层的ETL清洗服务。这也是我首次调部门的第一个项目。

## Kettle的介绍

Kettle从名称就可以知道“水壶”，顾名思义就是封装了很多内置功能，它是一个ETL的开源工具。Kettle就像一个容纳器，将很多ETL所需要的功能都涵盖进去，形成一个独立的生态系统。

### 功能介绍

下图可以看出,kettle包含了丰富的功能。

![Kettle功能图](https://note.youdao.com/yws/public/resource/9ad28471772957ddecb1e977ed1a2ac0/xmlnote/A8E422918CF347FFA48CDE9904FBBE0D/11674)

<html>
<center>kettle功能图</center>
</html>

### 为什么使用kettle

基于Kettle开发了一段时间，也逐渐深入了解到kettle的优势。以下是我使用Kettle以来个人的观点：

优势：
- 内置众多的成熟稳定开源组件，天然支持多种ETL业务场景。
- kettle设计是基于职责单一原则，各个组件独立，不相互依赖。因此提高重用性。
- 构建任务脚本简单，通过PC客户端拖拉组件的方式就能轻松实现转化任务。因此不用有编程基础的人都能够自行实现。
- kettle有独立的官网，上面文档较为完善，但是可能市面上不怎么流行，网上关于kettle的问题并不是很多。


劣势：
- 组件繁多，有额外的学习门槛。
- 没有提供Web端的开发页面，只有PC客户端，因此开发人员只能通过客户端调试并开发。
- 由于基于开源组件开发，调试难度相对较大。线上部署测试难度大。
- 天然包含众多组件，偏重量级开源框架，因此打包会比较大。
- 不支持对非结构化文本的解释，如非结构化网页html、爬虫、不支持对流的操作和传输、难以支持逻辑较为复杂的业务流程，如数据融合，数据分析等。
- 开发自定义组件相对复杂，调试比较难。

我大大小小也基于kettle开发过几十个脚本，但是往全公司推广确实举步维艰，因为需要学习成本，这个也跟公司的架构有关系（主要原因）。一旦脚本众多，维护起来就越来越麻烦，特别对于过于复杂的脚本来说，没几多人愿意维护起来。

### Kettle安装与使用

[Kettle安装使用](http://note.youdao.com/noteshare?id=28505b7348b9d73ed38d8da36bfd1425&sub=171D26FDD37442F184CA9B8947192B95)

### 集成Kettle

1. 添加Maven
```
<kettle.version>8.3.0.0-371</kettle.version>

<dependency>
	<groupId>pentaho-kettle</groupId>
	<artifactId>kettle-core</artifactId>
	<version>${kettle.version}</version>
</dependency>
<dependency>
	<groupId>pentaho-kettle</groupId>
	<artifactId>kettle-engine</artifactId>
	<version>${kettle.version}</version>
</dependency>
<dependency>
	<groupId>pentaho-kettle</groupId>
	<artifactId>kettle-dbdialog</artifactId>
	<version>${kettle.version}</version>
</dependency>
```

需要额外添加Pentaho自身的仓库，不然下载不了对应的Jar包
```
<repositories>
  	<repository>
      <id>pentaho-public</id>
      <name>Pentaho Public</name>
      <url>http://nexus.pentaho.org/content/groups/omni</url>
    </repository>
</repositories>
	
```

2. 代码集成

- 加载插件并进行kettle环境的初始化
```
@Override
public boolean init() {
	try {
		if (pluginDir != null) {
			// Load plugins
			StepPluginType.getInstance().getPluginFolders().add(new PluginFolder(pluginDir, false, true));
		}
		KettleEnvironment.init();
	} catch (KettleException e) {
		e.printStackTrace();
		Log.error("kettle executor init",e);
		return false;
	}
	return true;
}
```

- 执行Trans任务具体实现
```java
/**
 * 
 * @param filePath  脚本路径
 * @param paramMap  脚本参数
 * @param level 脚本输出日志级别
 * @return
 */
public boolean runTrans(String filePath, Map<String, String> paramMap,String level) {
		Trans trans = null;
		try {
			
			LOG.info("Running trans {}...", filePath);
			long st = System.currentTimeMillis();
			
			TransMeta transMeta = new TransMeta(filePath);    //构建Tran文件元数据对象
			
			trans = new Trans(transMeta);   //构建Trans任务

            //设置Trans里面输入的参数
			if (paramMap != null) {
				for( String param: paramMap.keySet() ) {
					trans.setParameterValue( param, paramMap.get(param));
					transMeta.setParameterValue( param, paramMap.get(param));
				}		
				trans.activateParameters();
			}
			//执行Trans任务的脚本
			trans.execute(null);
			trans.waitUntilFinished();
			
			if (trans.getErrors() > 0) {
				LOG.error("There are errors while running transformation! Error Code: {}", trans.getErrors());
				return false;
			} else {
				long dt = System.currentTimeMillis() - st;
				LOG.info("Run trans {} finish in {} ms", filePath, dt);
				return true;
			}
		} catch (Exception e) {
			e.printStackTrace();
			if(!trans.isFinished()){
				trans.stopAll();
			}
		} 
		return false;
	}
```

3. 运行

到了这部基本上已经可以跑kettle生成的trans的脚本，如果还想增加调度的功能话可以集成quartz调度包、市面上比较流行的saturn、xxl-job等调度框架，或者基于公司的业务架构来自行实现调度框架也是可以。以上的框架我也有使用过并进行对比和集成。后面会有一章节额外说明，这里就不详细说了。



## 参考

[kettle官网](https://community.hitachivantara.com/s/article/data-integration-kettle)
