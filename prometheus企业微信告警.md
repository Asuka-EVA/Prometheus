##### Prometheus配合 alertmanager 使用企业微信告警（坑已平！！！）





部署Prometheus 和 Alertmanager略



### **首先编写prometheus告警规则**

mkdir prometheus/rules/



vim prometheus/rules/host_monitor.yml



groups:

\- name: node-up

  rules:

  \- alert: node-up

​    expr: up == 0

​    for: 10s

​    labels:

​      severity: warning

​      team: node

​    annotations:

​      summary: "运维部门的 {{ $labels.instance }} 服务已停止运行超过 10s！"



#### 配置企业微信报警

首先使用企业微信创建一个企业

![img](assets/prometheus企业微信告警/1680004751417-f41a0e5c-7ee9-4e77-a044-d234cfa7d3d5.png)

然后点击头像，选择管理企业

![img](assets/prometheus企业微信告警/1680004751706-11484961-6cc7-4740-8349-ada4937a434a.png)



然后点击应用管理，选择创建应用

![img](assets/prometheus企业微信告警/1680004752015-81931120-c6d3-4c77-befc-fb3b2207f328.png)



![img](assets/prometheus企业微信告警/1680004752413-018c11cc-aba8-462d-877c-7a5c38ee2c74.png)









![img](assets/prometheus企业微信告警/1680004752683-93af646f-ddcf-4d11-a190-173cd861210b.png)

记录AgentId和Secret   

点击我的企业查看企业ID

![img](assets/prometheus企业微信告警/1680004752969-35a8ebce-f648-4eff-b1b1-350aa60e4f6e.png)

![img](assets/prometheus企业微信告警/1680004753222-900c4e6f-8b22-4177-b053-ed2841501a98.png)

查看组ID

![img](assets/prometheus企业微信告警/1680004753512-3d8c90fe-b739-4ae7-9597-af1b1b605d3f.png)

corp_id: 企业微信账号唯一 ID， 可以在我的企业中查看。

to_party: 需要发送的组(部门)。 可以在通讯录中查看

agent_id: 第三方企业应用的 ID 

api_secret: 第三方企业应用的密钥



企业id wwa9fa8a0ab8180c78

to_party 1

AgentId 1000002  

Secret lm-r9XLt8F4feyDonjDLLufx_dCjhcGr_w9xTHlObvE



然后编辑Alertmanager配置文件



vim alertmanager/alertmanager.yml

\# 定义路由树信息

route:

  group_by: ['alertname']	报警分组依据

  group_wait: 5s	最初即第一次等待多久时间发送一组警报的通知

  group_interval: 5m		在发送新警报前的等待时间

  repeat_interval: 5m		发送重复警报的周期 

  receiver: 'wechat'		发送警报的接收者的名称

receivers:

\- name: 'wechat'	和警报接收者名称一致

  wechat_configs:		企业微信配置

  \- corp_id: 'ww13cdaa9a2fa7e0af'

   to_party: '2'

   agent_id: '1000002'

   api_secret: 'F9LvwErjXM_Dn2IXr_nYEcvhgt1L-cBG9MmYX9QB3LI'

   send_resolved: true

   message: '{{ template "wechat.tmpl" . }}'



inhibit_rules: #抑制规则

  \- source_match: #源标签

​     severity: 'critical'

​    target_match:

​     severity: 'warning'

​    equal: ['alertname', 'dev', 'instance']

![img](assets/prometheus企业微信告警/1680004753843-ac731331-d920-48b4-bb4c-a0805fcb5c9c.png)



编写配置文件检测语法

./amtool check-config alertmanager.yml

![img](assets/prometheus企业微信告警/1680004754282-3e4017e2-ed22-47e6-9331-94d1398d7b28.png)

![img](assets/prometheus企业微信告警/1680004754740-2908354b-cb87-4951-a96c-cbad80e75067.png)

然后重启alertmanager 



systemctl restart alertmanager



systemctl stop node_exporter  停止node





然后发现并没有收到报警！！！经过排查发现



在配企业微信接收报警的时候发现多了一个企业可信ip

![img](assets/prometheus企业微信告警/1680004755133-db5131c9-9f29-4b73-ba88-11888a872135.png)



如果不配置这个的话没办法接收告警信息但是配了半天也没配成功，直接上配置流程。

![img](assets/prometheus企业微信告警/1680004755396-4abff05f-8ef3-4c33-b84e-b31dfc1ea839.png)

首先点开网页授权及JS-SDK 配置可信域名



![img](assets/prometheus企业微信告警/1680004755724-12c5e56e-bf37-4225-9d13-53b84e0db915.png)

点击申请校验域名后有一条提示 请下载文件WW_verify_z0kAmKeNiJVAwe5c点击下载

![img](assets/prometheus企业微信告警/1680004756021-3fff6241-b27d-4e95-a39b-2254565fda79.png)



然后来到阿里云官网  搜索函数计算

![img](assets/prometheus企业微信告警/1680004756307-2ab88134-c875-4ec8-9583-78f4de4b3e8d.png)

![img](assets/prometheus企业微信告警/1680004756744-4f1197df-5043-45e6-a5e3-818d66dc22b3.png)



新用户或者老用户都有免费试用，点击开通试用



开通试用后点击任务 然后创建函数

![img](assets/prometheus企业微信告警/1680004757061-71b7c940-ff76-48c4-a870-1d13e86b7fdb.png)

选择试用自定义运行

![img](assets/prometheus企业微信告警/1680004757378-9c5098af-8a04-4362-80d9-b2a0c0f4de9a.png)

然后

![img](assets/prometheus企业微信告警/1680004757684-8cb1ce56-e855-4f81-9415-24d751ccd4ad.png)

![img](assets/prometheus企业微信告警/1680004758315-54290aa0-c18c-445c-bfeb-5cdcde40f7a5.png)

![img](assets/prometheus企业微信告警/1680004758652-42ade835-2e18-4977-837f-114c457e639a.png)

创建完成后会自动跳转到创建的任务

![img](assets/prometheus企业微信告警/1680004759041-66d07c1f-51d8-4dc9-9d43-de68fd08dd0b.png)

![img](assets/prometheus企业微信告警/1680004759470-46f66c7e-2d51-457c-ae3c-914aeea009a1.png)

把第十七行的Hello, World!   换成 校验文件中的代码 z0k*****VAwe5c

![img](assets/prometheus企业微信告警/1680004759713-bb44d009-a1ea-41fc-991a-6edd4e6f3232.png)

![img](assets/prometheus企业微信告警/1680004759969-f5a031e5-9d7d-4fac-be6d-77900176c69f.png)

替换好以后点击部署代码

![img](assets/prometheus企业微信告警/1680004760420-f2f48b3e-c25e-4965-af41-b6262e94c374.png)

提示部署成功后点击     右侧的URL  然后复制这个链接

![img](assets/prometheus企业微信告警/1680004760706-28d3db43-2974-410d-9bfb-633b080cc800.png)

最后来到企业微信自建应用管理页面 把复制的链接去掉https的前缀  点击确定即可

![img](assets/prometheus企业微信告警/1680004761026-6dd92e34-c410-4bcd-a895-d0f2b0a8bf34.png)

可信域名配好之后就可以配置可信ip了

![img](assets/prometheus企业微信告警/1680004761402-79a68f18-e992-4503-b36c-d897aa0e872b.png)

然后测试一下报警功能发现正常了



![img](assets/prometheus企业微信告警/1680004761707-f771a498-0145-4751-9367-82628f39ac5b.png)