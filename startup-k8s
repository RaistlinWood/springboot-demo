#!/bin/bash

sleep 2
if !(which java 2>/dev/null); then
    echo '请安装java环境'
    exit
fi

PROJNAME=${PROJNAME}
APPNAME=${APPNAME}
ENV=${envID}
PROJ_APP=${PROJNAME}\_${APPNAME}
export JAVA_LOG_DIR=${JAVA_LOG_DIR:-"/service/logs/app/$ENV/$PROJNAME/$APPNAME"}

if [ ${#PROJ_APP} > 24 ];then
    PROJ_APP=${PROJ_APP:0:24}
fi

TIMESTAMP=`date +%Y%m%d%H%M%S`
agentID=$POD_IP

# -server -XX:+AlwaysPreTouch  # First should be default, but we make it explicit, second pre-zeroes memory mapped pages on JVM startup -- improves runtime performance
# G1 specific settings -- probably should be default for multi-core systems with >2 GB of heap (below that, default is probably fine)
# -XX:+UseG1GC
# -XX:+UseStringDeduplication
# -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=20  # Prevents G1 undersizing young gen, which otherwise causes a cascade of issues
# -XX:+ParallelRefProcEnabled # parallelize reference processing, reducing young and old GC times. We use a LOT of weak references, so should have big impact.
# -XX:+ExplicitGCInvokesConcurrent  # Avoid explicit System.gc() call triggering full GC, instead trigger G1

export MEM_OPTS="-server -Xss512k -XX:+AlwaysPreTouch -XX:+UseG1GC -XX:+UseStringDeduplication \
-XX:MaxMetaspaceSize=320M -XX:MaxGCPauseMillis=100 -XX:+ParallelRefProcEnabled \
-XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=25 -XX:+ExplicitGCInvokesConcurrent \
-XX:+UseContainerSupport -XX:InitialRAMPercentage=70.0 -XX:MaxRAMPercentage=70.0 "
export GC_OPTS="-Xloggc:${JAVA_LOG_DIR}/gc_${agentID}_${TIMESTAMP}.log \
-XX:ErrorFile=${JAVA_LOG_DIR}/hs_err_${agentID}_${TIMESTAMP}.log \
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCCause -XX:+PrintHeapAtGC \
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${JAVA_LOG_DIR}/heapdump_${agentID}_${TIMESTAMP}.hprof"

BASE_OPTS="-Djava.io.tmpdir=/tmp -Duser.timezone=Asia/Shanghai -Dfile.encoding=UTF-8"
# 数据库开启自动切换,jvm关闭 dns cache
DNS_OPTS="-Dsun.net.inetaddr.negative.ttl=1 -Dsun.net.inetaddr.ttl=0"

if [ "$envID" == "prod" ]; then
  START_OPTS="-Denv=PRO"
else
  START_OPTS="-Denv=${envID}"
fi

# IDC机房的ENV_ORIGIN传参，其他环境如AWS不支持 skywalking 与 fiss
ENV_ORIGIN_IDC=(dev sit uat sandbox prod)
if [[ ${ENV_ORIGIN_IDC[*]} =~ "${ENV_ORIGIN}" ]]; then
 START_OPTS="$START_OPTS -javaagent:/opt/lune/skywalking-agent/skywalking-agent.jar -javaagent:/opt/lune/fiss-agent.jar"
fi

exec java $MEM_OPTS $GC_OPTS $JMX_OPTS $START_OPTS $BASE_OPTS $DNS_OPTS -jar /opt/app.jar
