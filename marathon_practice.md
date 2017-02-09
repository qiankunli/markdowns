# marathon_practice

<!--
create time: 2017-02-09 10:21:53
Author: <TODO: 请写上你的名字>

This file is created by Marboo<http://marboo.io> template file $MARBOO_HOME/.media/starts/default.md
本文件由 Marboo<http://marboo.io> 模板文件 $MARBOO_HOME/.media/starts/default.md 创建
-->

## 实践的点

1. 为每个应用加label，添加服务类型、服务author等数据
2. xipam是否要建立保留ip区，以供数据库等服务使用
3. 清除不必要的数据。`docker sytem prune`（这个命令清除的很多，要留意）,`docker image prune`


## marathon.json	
	
	{
	  "id": "$JOB_NAME",
	  "cmd": null,
	  "cpus": 1,
	  "mem": 1024,
	  "disk": 0,
	  "instances": 1,
	  "labels":{"appType":"$APP_TYPE","author":"$BUILD_USER"}, # 通过标签标记应用的类型
	  "constraints": [
	  	["type", "CLUSTER","app-test"]
	  ],
	  "container": {
	    "type": "DOCKER",
	    "volumes": [
	      {
	        "containerPath": "/logs",
	        "hostPath": "/var/log/app",
	        "mode": "RW"
	      }
	    ],
	    "docker": {
	      "image": "test/$JOB_NAME:$BUILD_NUMBER",
	      "network": "USER",
	      "privileged": false,
	      "forcePullImage": true,
	      "parameters": [		# 指定docker特定的运行参数
	      	   { "key": "network", "value": "macvlan96" }
	      ]
	    }
	  },
	  "ipAddress": {
		"networkName": "macvlan96"
	  },
	  "healthChecks": [{
	    "port": 8080,
	    "protocol": "TCP",
	    "gracePeriodSeconds": 180,
	    "intervalSeconds": 15,
	    "timeoutSeconds": 30,
	    "maxConsecutiveFailures": 3  # 失败检查重试次数，过次数后认为不可用
	  }],
	  "upgradeStrategy": {
	    "minimumHealthCapacity": 0,
	    "maximumOverCapacity": 1
	  }
	}




mesos 有关于 ip-per-task的论述，但里面提到的ipam是mesos的moudle。
