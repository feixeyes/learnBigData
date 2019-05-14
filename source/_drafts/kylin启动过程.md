---
title: kylin启动过程
tags:
---

    #To save all these troubles, we use hbase runjar to start tomcat.
    #In this way we no longer need to explicitly configure hadoop/hbase related classpath for tomcat,
    #hbase command will do all the dirty tasks for us:


     hbase
     org.apache.hadoop.util.RunJar
     ${tomcat_root}/bin/bootstrap.jar  org.apache.catalina.startup.Bootstrap start

