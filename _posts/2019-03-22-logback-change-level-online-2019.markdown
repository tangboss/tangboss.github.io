---
layout:     post
title:      "logback动态调整打印级别"
subtitle:   " \"change log level online\""
date:       2019-03-22 19:12:00
author:     "tangboss"
tags:
    - logback
    - scala
    - log level
    - 
---


## 需求

关于日志级别修改，目前基本都要重启服务生效，很多次查问题时，重启后故障问题就恢复了，再复现比较困难。

具体针对于使用logback作为日志记录的有两种方法：

1）封装logger接口，通过restful接口修改打印级别。

2）修改外置logback.xml文件，通过logback的周期性扫描修改生效。

---

## restful接口修改

### 接口封装

对于使用的logger.info进行再封装，将打印级别外在内存中，由restful接口触发修改
    
    case class LogSwitch(logType:String, mode:String)
 
 
	object LogSwitchService {
	  val logSwitchMap =  mutable.Map[String, String]("info" -> "on",
	                                                  "error" -> "on",
	                                                  "debug" -> "off"
	  )
	 
	  def set(logSwitch: LogSwitch) = {
	    if(logSwitchMap.get(logSwitch.logType).isDefined){
	      logSwitchMap += (logSwitch.logType -> logSwitch.mode)
	    }
	    Ok(JsonMapper.to(logSwitchMap))
	  }
	 
	  def get(logType: String):String = {
	    logSwitchMap.getOrElse(logType, "off")
	  }
	}
	 
	object LogService {
	  var printLog = false
	  val logger = LoggerFactory.getLogger("debug")
	 
	  def info(str:String, logType:String="info") {
	    if(printLog) println("Log: " + str)
	    if("on" == LogSwitchService.get(logType)) logger.info(str)
	  }
	 
	  def error(str:String, logType:String="error") {
	    if("on" == LogSwitchService.get(logType)) logger.error(str)
	  }
	 
	  def debug(str:String, logType:String="debug") {
	    if("on" == LogSwitchService.get(logType)) logger.info(str)
	  }
	}

### 动态调整
	LogService.info("------------------string---------")
	LogService.debug("------------------string---------")
	
	POST /api/v1.0/logSwitch
	{"logType":"debug","mode":"off"}

---

## logback.xml在线修改

### 机制
    
   修改logback.xml配置文件：在configuration根节点配置属性scan和scanPeriod，scan为true时，配置文件被修改会被重新加载，scanPeriod定义了扫描文件变化的周期，默认60000毫秒，即一分钟。

    <configuration scan="true" scanPeriod="60 seconds" debug="true">


### logback.xml外置

    override def start(path: String, port: String) = {
	  loadLogbackConfig("./logback.xml")  //logback.xml的文件放置在与war包同目录
	  super.start(path, port)
	}
	 
	private def loadLogbackConfig(externalFileName: String="") {
	  val externalConfigFile: File = new File(externalFileName)
	  if (externalConfigFile.canRead) {
	    val lc: LoggerContext = LoggerFactory.getILoggerFactory.asInstanceOf[LoggerContext]
	    val configurator: JoranConfigurator = new JoranConfigurator
	    configurator.setContext(lc)
	    lc.reset()
	    configurator.doConfigure(externalConfigFile.getPath)
	    StatusPrinter.printInCaseOfErrorsOrWarnings(lc)
	  }

### 级别动态修改

logback.xml放置在war包同目录下，此时可修改级别，可以将INFO级别修改为DEBUG，等待扫描周期（1分钟）后，可看到程序重新加载logback.xml。

	<root level="INFO"> -----> <root level="DEBUG">
	<!--<appender-ref ref="STDOUT"/>-->
	<appender-ref ref="ALLFILE"/>
	<appender-ref ref="ERROFILE"/>
	</root>
	</configuration>



检测输出

	17:13:06,289 |-INFO in ReconfigureOnChangeTask(born:1553245506287) - Detected change in configuration files.
	17:13:06,290 |-INFO in ReconfigureOnChangeTask(born:1553245506287) - Will reset and reconfigure context named [default]
	17:13:06,290 |-INFO in ch.qos.logback.classic.jmx.JMXConfigurator(default) - onReset() method called JMXActivator [ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator]
	17:13:06,289 |-INFO in ReconfigureOnChangeTask(born:1553245506287) - Detected change in configuration files.
	17:13:06,290 |-INFO in ReconfigureOnChangeTask(born:1553245506287) - Will reset and reconfigure context named [default]
	17:13:06,290 |-INFO in ch.qos.logback.classic.jmx.JMXConfigurator(default) - onReset() method called JMXActivator [ch.qos.logback.classic:Name=default,Type=ch.qos.logback.classic.jmx.JMXConfigurator]
	17:13:06,295 |-INFO in ch.qos.logback.classic.joran.action.ConfigurationAction - Will scan for changes in [file:/root/code/./logback.xml]

---

## logback+restful

### 机制

logback除了提供通过配置文件修改，还可提供setLevel/getLevel函数进行内存打印级别的修改。

### 实现


	put("/log"){
	  val dynLogger = LoggerFactory.getLogger(org.slf4j.Logger.ROOT_LOGGER_NAME)
	    .asInstanceOf[ch.qos.logback.classic.Logger]
	  if(request.getParameter("level") != null){
	    val level = request.getParameter("level").toUpperCase match{
	      case "DEBUG" => Level.DEBUG
	      case "INFO" => Level.INFO
	      case "ERROR" => Level.ERROR
	      case _ => Level.INFO
	    }
	    dynLogger.setLevel(level)
	  }
	  Ok(JsonMapper.to(dynLogger.getLevel))
	}

### 设置

接口 PUT http://ip:port/api/v1.0/log?level=info/debug/error

返回值：为当前打印级别  {"levelInt":10000,"levelStr":"DEBUG"}

1）若要查询当前打印级别可直接不带level：PUT http://ip:port/api/v1.0/log

2）若传输level字符不匹配，默认使用info级别