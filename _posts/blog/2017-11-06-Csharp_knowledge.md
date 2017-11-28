---
layout: post
title: c# sharp 基础
category: blog
description: 介绍C#开发中会普遍用到的日志库log4net
---

# c# sharp 基础

## 引

最近看了一个视频，谈到学习一个新知识练习20小时就可以达到能够做一些东西。
算下来，我从研究生时期接触C#语言，到现在也写过几个小软件了。但也没有达到可以流畅的出一些自己觉得满意的东西。
最近因为工作的原因又把这个东西捡起来了，开始开发一些桌面软件，所以打算再花上20个小时来精进一下这个技能。
这篇blog初衷便与此，记录下自己觉得比较重要的知识。以后用到的时候来这里查就好。


## log4net

log4net是一个功能著名的开源日志记录组件。利用log4net可以方便地将日志信息记录到文件、控制台、Windows事件日志和数据库（包括MS SQL Server, Access, Oracle9i,Oracle8i,DB2,SQLite）中。
并且我们还可以记载控制要记载的日志级别，可以记载的日志类别包括：FATAL（致命错误）、ERROR（一般错误）、WARN（警告）、INFO（一般信息）、DEBUG（调试信息）。
要想获取最新版本的log4net组件库，可以到官方网站http://logging.apache.org/log4net/下载。

### 使用方法
1. 在程序中添加log4net.dll的引用(这个dll文件可以在官网上下载)
2. 配置app.config 文件
  <?xml version="1.0" encoding="utf-8" ?>
  <configuration>
    <configSections>
      <section name="log4net" type="log4net.Config.Log4NetConfigurationSectionHandler, log4net"/>
    </configSections>

    <log4net>
      <appender name="RollingLogFileAppender" type="log4net.Appender.RollingFileAppender">
        <!--日志路径-->
        <param name= "File" value= "D:\App_Log\"/>
        <!--是否是向文件中追加日志-->
        <param name= "AppendToFile" value= "true"/>
        <!--log保留天数-->
        <param name= "MaxSizeRollBackups" value= "10"/>
        <!--日志文件名是否是固定不变的-->
        <param name= "StaticLogFileName" value= "false"/>
        <!--日志文件名格式为:2008-08-31.log-->
        <param name= "DatePattern" value= "yyyy-MM-dd&quot;.log&quot;"/>
        <!--日志根据日期滚动-->
        <param name= "RollingStyle" value= "Date"/>
        <layout type="log4net.Layout.PatternLayout">
          <param name="ConversionPattern" value="%d [%t] %-5p %c - %m%n %loggername" />
        </layout>
      </appender>

      <!-- 控制台前台显示日志 -->
      <appender name="ColoredConsoleAppender" type="log4net.Appender.ColoredConsoleAppender">
        <mapping>
          <level value="ERROR" />
          <foreColor value="Red, HighIntensity" />
        </mapping>
        <mapping>
          <level value="Info" />
          <foreColor value="Green" />
        </mapping>
        <layout type="log4net.Layout.PatternLayout">
          <conversionPattern value="%n%date{HH:mm:ss,fff} [%-5level] %m" />
        </layout>

        <filter type="log4net.Filter.LevelRangeFilter">
          <param name="LevelMin" value="Info" />
          <param name="LevelMax" value="Fatal" />
        </filter>
      </appender>

      <root>
        <!--(高) OFF > FATAL > ERROR > WARN > INFO > DEBUG > ALL (低) -->
        <level value="all" />
        <appender-ref ref="ColoredConsoleAppender"/>
        <appender-ref ref="RollingLogFileAppender"/>
      </root>
    </log4net>
  </configuration>
  
  3. 在需要引入 log4net 的代码文件中，添加代码。例如下面是 main.cs 的写法

    using System;
    using System.Collections.Generic;
    using System.Text;
    using System.Windows.Forms;
    using System.Reflection;
    using log4net;

    //注意下面的语句一定要加上，指定log4net使用.config文件来读取配置信息
    [assembly: log4net.Config.XmlConfigurator(Watch = true)]
    namespace Log4NetDemo
    {
        /// <summary>
        /// 说明：本程序演示如何利用log4net记录程序日志信息。log4net是一个功能著名的开源日志记录组件。
        /// 利用log4net可以方便地将日志信息记录到文件、控制台、Windows事件日志和数据库中（包括MS SQL Server, Access, Oracle9i,Oracle8i,DB2,SQLite）。
        /// 下面的例子展示了如何利用log4net记录日志

        public class MainClass
        {
            public static void Main(string[] args)
            {
                //Application.Run(new MainForm());
                //创建日志记录组件实例
                ILog log = log4net.LogManager.GetLogger(MethodBase.GetCurrentMethod().DeclaringType);
                //记录错误日志
                log.Error("error",new Exception("发生了一个异常"));
                //记录严重错误
                log.Fatal("fatal",new Exception("发生了一个致命错误"));
                //记录一般信息
                log.Info("info");
                //记录调试信息
                log.Debug("debug");
                //记录警告信息
                log.Warn("warn");
                Console.WriteLine("日志记录完毕。");
                Console.Read();
            }
        }
    }
    
用法如此了，非常简单，总的来说，工作量集中在配置文件的撰写上。

## 程序打包发布


