# 部署prometheus+grafana+alertmanager+钉钉监控中心

## 部署前准备：

下载二进制安装文件   https://prometheus.io/download/

**官方组件：**

| 组件                                                         | 端口 |
| ------------------------------------------------------------ | ---- |
| prometheus                                                   | 9090 |
| node_exporter                                                | 9111 |
| blackbox_exporter                                            | 9115 |
| alertmanager                                                 | 9093 |
| [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics) | 8428 |

**第三方开源软件：**

| 组件                                                         | 端口 |
| ------------------------------------------------------------ | ---- |
| [ConsulManager](https://github.com/starsliao/ConsulManager)  |      |
| [PrometheusAlert](https://github.com/feiyu563/PrometheusAlert) |      |

**部署环境：**

centos 7 

**部署方式：**

官方组件全部采用二进制部署，第三方开源组软件采用docker方式部署

## 安装步骤

\1. **官网下载官方组件，全部解压至/data**

\2. **修改文件名，去掉后面版本号**

Gomv prometheus-2.39.1.linux-amd64 prometheus

\3. **配置系统服务**

**prometheus：**

Gocat > /etc/systemd/system/prometheus.service <<EOF [Unit] Description=prometheus After=network.target [Service] Type=simple User=prometheus ExecReload=/bin/kill -HUP \$MAINPID ExecStart=/data/prometheus/prometheus --config.file=/data/prometheus/prometheus.yml --storage.tsdb.path=/data/prometheus/data --storage.tsdb.retention.time=2h --web.enable-lifecycle Restart=on-failure [Install] WantedBy=multi-user.target EOF

Go#刷新配置文件，并设置开机自启动

**node_exporter节点监控**

Gocat > /etc/systemd/system/node_exporter.service <<EOF [Unit] Description=node_exporter After=network.target [Service] Type=simple User=prometheus ExecStart=/data/node_exporter/node_exporter  --web.listen-address=:9111 Restart=on-failure [Install] WantedBy=multi-user.target EOF



Go

**alertmanager**

Gocat > /etc/systemd/system/alertmanager.service <<EOF [Unit] Description=alertmanager After=network.target [Service] Type=simple User=prometheus ExecStart=/data/alertmanager/alertmanager --config.file=/data/alertmanager/alertmanager.yml --storage.path=/data/alertmanager/data --cluster.advertise-address=0.0.0.0:9093 Restart=on-failure [Install] WantedBy=multi-user.target EOF



Go

**blackbox_exporter**

Gocat > /etc/systemd/system/blackbox_exporter.service<<EOF [Unit] Description=blackbox_exporter After=network.target [Service] Type=simple User=prometheus ExecStart=/data/blackbox_exporter/blackbox_exporter --config.file=/data/blackbox_exporter/blackbox.yml Restart=on-failure [Install] WantedBy=multi-user.target EOF



Go

**vmstorage**

Gocat > /etc/systemd/system/vmstorage.service<<EOF [Unit] Description=vmstorage After=network.target [Service] Type=simple User=root ExecStart=/data/victoria-metrics-prod -storageDataPath=/data/vmdata -retentionPeriod=12 -search.maxQueryLen=65535 Restart=on-failure LimitNOFILE=1058576 [Install] WantedBy=multi-user.target EOF



Go

将以下行添加到 Prometheus 配置文件（通常位于/etc/prometheus/prometheus.yml），以便将数据发送到 VictoriaMetrics：

Goremote_write:   - url: http://<victoriametrics-addr>:8428/api/v1/write

替换<victoriametrics-addr>为 VictoriaMetrics 的主机名或 IP 地址。然后通过以下命令应用新配置：

kill -HUP `pidof prometheus`

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278626868-8e6818a3-8f9a-4027-a456-101835811392.png)

Prometheus 将传入的数据写入本地存储并并行复制到远程存储。这意味着--storage.tsdb.retention.time即使远程存储不可用，数据在本地存储中仍然可用。

如果您计划从多个 Prometheus 实例向 VictoriaMetrics 发送数据，则将以下行添加到[Prometheus config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#configuration-file)global部分：

Goglobal:   external_labels:     datacenter: dc-123

这指示 Prometheus 在将datacenter=dc-123每个样本发送到远程存储之前为其添加标签。标签名称可以是任意的 -datacenter只是一个示例。标签值在 Prometheus 实例中必须是唯一的，因此可以按此标签过滤和分组时间序列。

对于高负载的 Prometheus 实例（每秒 200k+ 样本），可以应用以下调优：

Goremote_write:   - url: http://<victoriametrics-addr>:8428/api/v1/writequeue_config:       max_samples_per_send: 10000capacity: 20000max_shards: 30

使用远程写入可将 Prometheus 的内存使用量增加约 25%。如果您遇到 Prometheus 内存消耗过高的问题，请尝试降低max_samples_per_send和capacity参数。请记住，这两个参数紧密相连。[在此处](https://prometheus.io/docs/practices/remote_write)阅读有关为 Prometheus 调整远程写入的更多信息。

建议将 Prometheus 升级到[v2.12.0](https://github.com/prometheus/prometheus/releases)或更高版本，因为以前的版本可能存在remote_write.

还可以看看[vmagent](https://docs.victoriametrics.com/vmagent.html) 和[vmalert](https://docs.victoriametrics.com/vmalert.html)，它们可以用作 Prometheus 的更快、更少资源消耗的替代品。

**consul**

**安装**

Go# 使用yum部署consul yum install -y yum-utils yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo yum -y install consul # 或者直接下RPM包安装(推荐) wget https://rpm.releases.hashicorp.com/RHEL/7/x86_64/stable/consul-1.12.2-1.x86_64.rpm rpm -ivh ./consul-1.12.2-1.x86_64.rpm

**配置**

Govim /etc/consul.d/consul.hcl log_level = "ERROR" #日志级别，日志太多可以只打印error日志，不需要可以去掉这行。 advertise_addr = "10.100.14.23" #填写你的网卡IP，如果启动或运行有报错，可以尝试去掉这行。 data_dir = "/data/consul" client_addr = "0.0.0.0" ui_config{   enabled = true } server = true bootstrap = true acl = {   enabled = true   default_policy = "deny"   enable_token_persistence = true }

**启动服务**

Gochown -R consul:consul /data/consul  #注意下数据目录的权限。 systemctl enable consul.service systemctl start consul.service

**获取登录Token（记录SecretID，即为Consul登录的Token）**

Goroot@localhost:/data 16:41:13 > consul acl bootstrap AccessorID:       74b11313-a37b-8366-3039-9960d3a628b0 SecretID:         17f5f37d-bc89-1431-732e-a9ea4536a53a Description:      Bootstrap Token (Global Management) Local:            false Create Time:      2022-11-07 16:41:21.171293265 +0800 CST Policies:    00000000-0000-0000-0000-000000000001 - global-management

**ConsulManager**

**💾部署说明**

\1. ConsulManager需要依赖Consul，请先完成Consul的部署。（[docs/Consul部署说明.md](https://github.com/starsliao/ConsulManager/blob/main/docs/Consul部署说明.md)）

\2. 使用docker-compose来部署ConsulManager

下载：wget https://starsl.cn/static/img/docker-compose.yml（仓库根目录下docker-compose.yml）

编辑：docker-compose.yml，修改3个环境变量：

consul_token：consul的登录token（[如何获取？](https://github.com/starsliao/ConsulManager/blob/main/docs/Consul部署说明.md#获取登录token记录secretid即为consul登录的token)）

consul_url：consul的URL(http开头，/v1要保留)

admin_passwd：登录ConsulManager Web的admin密码

启动：docker-compose pull && docker-compose up -d

访问：http://{IP}:1026，使用配置的ConsulManager admin密码登录

**PrometheusAlert**

Go#打开PrometheusAlert releases页面，根据需要选择需要的版本下载到本地解压并进入解压后的目录 如linux版本(https://github.com/feiyu563/PrometheusAlert/releases/download/v4.7/linux.zip)  # wget https://github.com/feiyu563/PrometheusAlert/releases/download/v4.7/linux.zip && unzip linux.zip &&cd linux/  #运行PrometheusAlert # ./PrometheusAlert (#后台运行请执行 nohup ./PrometheusAlert &)  #启动后可使用浏览器打开以下地址查看：http://127.0.0.1:8080 #默认登录帐号和密码在app.conf中有配置

**grafana**

Gowget https://dl.grafana.com/oss/release/grafana-9.2.3-1.x86_64.rpm  # sudo yum install grafana-9.2.3-1.x86_64.rpm rpm -ivh xxxxxxxxxxxxxxx systemctl start grafana-server.service && systemctl enable grafana-server.service && systemctl status grafana-server.service

**prometheus-webhook-dingtalk**

这是实现钉钉告警的第三方插件

C#cat /usr/lib/systemd/system/prometheus-webhook.service  [Unit] Description=Prometheus Dingding Webhook [Service] ExecStart=/data/dingtalk-webhook/prometheus-webhook-dingtalk --config.file=/data/dingtalk-webhook/config.yml ExecReload=/bin/kill -HUP $MAINPID KillMode=process Restart=on-failure [Install] WantedBy=multi-user.target

**使用prometheus监控主机**

\1. **在被监控主机上安装node exporter**

Gocd /data/ tar -xvf node_exporter-1.4.0.linux-amd64 mv node_exporter-1.4.0.linux-amd64 node_exporter cat > /etc/systemd/system/node_exporter.service <<EOF [Unit] Description=node_exporter After=network.target [Service] Type=simple User=prometheus ExecStart=/data/node_exporter/node_exporter  --web.listen-address=:9111 Restart=on-failure [Install] WantedBy=multi-user.target EOF



Go#加载配置 systemctl daemon-reload #启动node exporter systemctl start node_exporter #检查状态 systemctl status node_exporter

\2. **配置Prometheus，以收集node exporter的数据**

我们需要在prometheus服务器中配置，让prometheus去pull这个被监控主机的数据。

Govim /data/prometheus/prometheus.yml #在scrape_configs:模块下添加以下配置： #采集node exporter监控数据  - job_name: 'linux-node'     static_configs:       - targets: ['10.100.14.4:9112']         labels:           instance: "k8s-dev-gw"

将监控prometheus默认的localhost改为prometheus服务器ip：

修改后示例：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278627465-9abb919a-11c9-4c7a-b1b2-cb177be6717e.png)

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278627943-c28bbc47-2c63-4bf6-96b4-b839f688b89a.png)

\3. **加载配置文件**

Go#检查配置文件语法 root@localhost:/data/prometheus 13:19:22 > ./promtool check config  /data/prometheus/prometheus.yml Checking /data/prometheus/prometheus.yml  SUCCESS: /data/prometheus/prometheus.yml is valid prometheus config file syntax #加载配置文件，这样就不用重启prometheus了root@localhost:/data/prometheus 13:19:43 > curl -X POST http://10.100.14.23:9090/-/reload

此时在Prometheus web界面上就能看到被监控的主机了：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278628394-968980de-85db-43f3-9ebe-28a05e3961e3.png)

**配置Prometheus**

**1. 配置Prometheus与Alertmanager通信**

 

打开prometheus的配置文件 /data/prometheus/prometheus.yml , /data/prometheus/prometheus2.yml 分别进行修改（根据实际情况,有两个prometheus实例）
 找到并修改为如下内容：

Objective-C# Alertmanager configuration alerting:   alertmanagers:   - static_configs:     - targets:       - 127.0.0.1:9093        # Load rules once and periodically evaluate them according to the global 'evaluation_interval'. rule_files:   # - "first_rules.yml"   # - "second_rules.yml"- "rules/*.yml"

**2. 创建规则目录**

Plain Textmkdir -pv /data/prometheus/rules

**3. 配置规则**

 

注意： 这里列举两个常用的规则文件，其余根据实际情况自行修改(可以去prometheus的web页面上自己先查一遍，看表达式是否正确查出数据)

**a. 主机存活告警文件,分组名为servers_survival：**

Bashroot@rancher2x.hw:/data/prometheus/rules# cat servers_survival.yml  groups: - name: servers_survival   rules:   - alert: 节点存活--测试--应用服务器   #告警规则名称     expr:  up{job="hw-nodes-test-rancher"} == 0     for: 1m    #等待评估时间     labels:    #自定义标签，定义一个level标签，标记这个告警规则警告级别: critical严重,warning警告       level: critical     annotations:   #指定附加信息（邮件标题文本）       summary: "机器 {{ $labels.instance }} 挂了"       description: "服务器{{$labels.instance}} 挂了 (当前值: {{ $value }})"    - alert: 节点存活--华为云--生产其他服务器     expr:  up{job="hw-nodes-prod-other"} == 0     for: 1m       labels:       level: critical     annotations:       summary: "机器 {{ $labels.instance }} 挂了"       description: "{{$labels.instance}} 宕机(当前值: {{ $value }})"    - alert: 节点存活--华为云--生产ES服务器        expr:  up{job="hw-nodes-prod-ES"} == 0     for: 1m         labels:           level: critical     annotations:          summary: "机器 {{ $labels.instance }} 挂了"       description: "{{$labels.instance}} 宕机(当前值: {{ $value }})"

**b. 主机状态告警文件,分组名为servers_status：**

Kotlinroot@rancher2x.hw:/data/prometheus/rules# cat servers_status.yml  groups:- name: servers_status   rules:- alert: CPU负载1分钟告警     expr:  node_load1{job!~"(nodes-dev-GPU|hw-nodes-test-server|hw-nodes-prod-ES|hw-nodes-prod-MQ)"} / count (count (node_cpu_seconds_total{job!~"(nodes-dev-GPU|hw-nodes-test-server|hw-nodes-prod-ES|hw-nodes-prod-MQ)"}) without (mode)) by (instance, job) > 2.5for: 1m       labels:       level: warning     annotations:       summary: "{{ $labels.instance }} CPU负载告警 "       description: "{{$labels.instance}} 1分钟CPU负载(当前值: {{ $value }})"- alert: CPU使用率告警     expr:  1 - avg(irate(node_cpu_seconds_total{mode="idle",job!~"(IDC-GPU|hw-nodes-prod-ES|nodes-test-GPU|nodes-dev-GPU)"}[30m])) by (instance) > 0.85for: 1m       labels:       level: warning     annotations:       summary: "{{ $labels.instance }} CPU使用率告警 "       description: "{{$labels.instance}} CPU使用率超过85%(当前值: {{ $value }} )"- alert: CPU使用率告警      expr: 1 - avg(irate(node_cpu_seconds_total{mode="idle",job=~"(IDC-GPU|hw-nodes-prod-ES)"}[30m])) by (instance) > 0.9for: 1m     labels:       level: warning     annotations:       summary: "{{ $labels.instance }} CPU负载告警 "       description: "{{$labels.instance}} CPU使用率超过90%(当前值: {{ $value }})"- alert:  内存使用率告警     expr:  (1-node_memory_MemAvailable_bytes{job!="IDC-GPU"} /  node_memory_MemTotal_bytes{job!="IDC-GPU"}) * 100 > 90     labels:       level: critical     annotations:       summary: "{{ $labels.instance }} 可用内存不足告警"       description: "{{$labels.instance}} 内存使用率已达90% (当前值: {{ $value }})"- alert:  磁盘使用率告警     expr: 100 - (node_filesystem_avail_bytes{fstype=~"ext4|xfs", mountpoint !~ "/var/lib/[kubelet|rancher].*" } / node_filesystem_size_bytes{fstype=~"ext4|xfs", mountpoint !~ "/var/lib/[kubelet|rancher].*"}) * 100 > 85     labels:       level: warning     annotations:       summary: "{{ $labels.instance }} 磁盘使用率告警"       description: "{{$labels.instance}} 磁盘使用率已超过85% (当前值: {{ $value }})"

**4. 热加载prometheus配置**

C++curl -X POST http://localhost:9090/-/reload

 

去web上查看确认rules是否被prometheus加载

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278629069-281b7cc7-e043-49fa-a0e3-8a80e1188776.png)

**配置Alertmanger**

 

上面配置了prometheus与alertmanager的通信，接下来我们配下alertmanager来实现发送告警信息给我们
 这里我们主要以钉钉告警为例子

**1. 钉钉上添加一个钉钉机器人,**

 

设置好名字，群组，选择加签，确定。

**2. 修改prometheus-webook配置文件绑定申请的机器人**

 

我只绑定了一个webhook所以只要配置到webhook1

Bash cat /data/dingtalk-webhook/config.yml  ## Customizable templates path templates: ##  - templates/alertmanager-dingtalk.tmpl   - /data/dingtalk-webhook/dingding3.tmpl # 配置告警模板的所在位置  #default_message: #  title: '{{ template "legacy.title" . }}' #  text: '{{ template "legacy.content" . }}'  ## Targets, previously was known as "profiles" targets:   webhook1:     url: https://oapi.dingtalk.com/robot/send?access_token=xxxxxxxxxx  # 配置机器人的webhook_url     # secret for signature     secret: SEC65342be21ab54b730da9347be9307b7831bd65adf1c99406fedc786f62fecb98 # 配置加签（申请的时候那串数字）     message:       title: '{{ template "ops.title" . }}'  #  给这个webhook应用上 模板标题 （ops.title是我们模板文件中的title 可在下面给出的模板文件中看到）       text: '{{ template "ops.content" . }}' #  给这个webhook应用上 模板内容  （ops.content是我们模板文件中的content 可在下面给出的模板文件中看到）

**3. 告警模板文件**

Ruby cat /data/dingtalk-webhook/dingding3.tmpl     {{ define "__subject" }} [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ end }}     {{ define "__alert_list" }}{{ range . }} ---     **告警类型**: {{ .Labels.alertname }}      **告警级别**: {{ .Labels.level }}     **故障主机**: {{ .Labels.instance }}      **告警信息**: {{ .Annotations.description }}     **触发时间**: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}   {{ end }}{{ end }}   {{ define "__resolved_list" }}{{ range . }} ---     **告警类型**: {{ .Labels.alertname }}      **告警级别**: {{ .Labels.level }}     **故障主机**: {{ .Labels.instance }}      **触发时间**: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}       **恢复时间**: {{ (.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }} {{ end }}{{ end }}     {{ define "ops.title" }} {{ template "__subject" . }} {{ end }}   {{ define "ops.content" }} {{ if gt (len .Alerts.Firing) 0 }} **====侦测到{{ .Alerts.Firing | len  }}个故障====** {{ template "__alert_list" .Alerts.Firing }} --- {{ end }}   {{ if gt (len .Alerts.Resolved) 0 }} **====恢复{{ .Alerts.Resolved | len  }}个故障====** {{ template "__resolved_list" .Alerts.Resolved }} {{ end }} {{ end }}   {{ define "ops.link.title" }}{{ template "ops.title" . }}{{ end }} {{ define "ops.link.content" }}{{ template "ops.content" . }}{{ end }} {{ template "ops.title" . }} {{ template "ops.content" . }} 

**4. 修改alertmanager配置文件为如下内容**

 

注：这里也加上了邮件相关的配置

C# cat alertmanager.yml      global:   resolve_timeout: 5m   smtp_smarthost: 'smtp.163.com:25'   smtp_from: 'xxx@163.com'   smtp_auth_username: 'xxxx@163.com'   smtp_auth_password: '邮箱的授权码'   smtp_require_tls: false templates:- '/data/dingtalk-webhook/*.tmpl'  #告警模板位置 route:   group_by: ['servers_survival','servers_status'] # 根据告警规则组名进行分组   group_wait: 30s # 分组内第一个告警等待时间，10s内如有第二个告警会合并一个告警   group_interval: 5m  # 发送新告警间隔时间   repeat_interval: 30m #重复告警间隔发送时间，如果没处理过多久再次发送一次   receiver: 'dingtalk_webhook' # 接收人  receivers:- name: 'ops'    email_configs:- to: 'tianye@163.com'      html: '{{ template "email.to.html" .}}'      headers: { Subject: "[WARNING]Prometheus告警邮件" }      send_resolved: true- name: 'dingtalk_webhook'    webhook_configs:- url: 'http://localhost:8060/dingtalk/webhook1/send' # 填写prometheus-webhook的webhook1 url      send_resolved: true     # 在恢复后是否发送恢复消息给接收人

**5. 重启服务**

CSSsystemctl restart prometheus-webhook.service  systemctl restart alertmanager.service 

**使用ConsulManager实现批量主机接入**

**prometheus接入步骤：**

登陆ConsulManager控制台，依次点击云资源管理----->>>ECS管理----->>>自建主机主机管理（根据实际情况填写）：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278629563-96487f1e-7285-4ffe-86af-ecbacaa92c45.png)

在上图点击导出即可导出模板，根据自身详细信息填写模板：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278630423-778cb851-2eca-4829-a8ad-3cfbf6c0eeb8.png)

填写完成保存文件，回到控制台依次点击导入----->>>选择文件----->>>打开，即可看到服务器信息已经导入到控制台：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278631956-39ec09c0-1250-45e6-a2da-622677204c76.png)

导入完成后，依次点击prometheus配置----->>>选择服务器组+操作系统----->>>生成配置----->>>复制配置：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278633193-14883fe1-8964-4770-8a03-c7213514d66b.png)

回到命令行界面，将刚刚复制的配置添加进/data/prometheus/prometheus.yaml里，格式不用自己改，consul-manager已经做过处理了：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278634730-4432f9d9-1981-489a-99e5-fc64063e1899.png)

进入http://10.100.14.23:9090/targets，prometheus的targets控制台即可看到刚刚添加的job以及服务器信息：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278635765-3f575176-2bb6-409e-bcf3-325ae4a91668.png)

**告警配置**

继续回到consulmanager控制台页面，点击告警规则，复制配置：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278636576-0f958a0e-bf2e-4a3a-8f2a-3f344a4a33c4.png)

将刚刚复制的配置文件加入rules目录下：vim node_exporter.yml

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278638927-de01517f-6c00-4b39-9597-e61db3830337.png)

热加载配置文件：

Gocurl -X POST http://localhost:9090/-/reload

此时打开http://10.100.14.23:9090/alerts，prometheus的alerts页面即可看到刚刚加入的告警配置文件：

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278640002-f7522231-b766-4584-9472-d4e6d2b1024a.png)

**使用prometheus监控gitlab**

需要进行一些配置更改，才能由外部 Prometheus 服务器监视 GitLab. 对于[具有多个节点的 GitLab 部署，](https://www.kancloud.cn/reference_architectures/index.html)建议使用外部服务器.

**编辑配置文件**

要使用外部 Prometheus 服务器：

Govim /etc/gitlab/gitlab.rb

\1. **禁用捆绑的 Prometheus：**

Plaintextprometheus['enable'] = false 

\2. **将每个捆绑服务的**[**导出器设置**](https://www.kancloud.cn/apachecn/gitlab-doc-zh/1949041#bundled-software-metrics)**为侦听网络地址，例如：**

Plaintextgitlab_exporter['listen_address'] = '0.0.0.0' sidekiq['listen_address'] = '0.0.0.0' gitlab_exporter['listen_port'] = '9168' node_exporter['listen_address'] = '0.0.0.0:9100' redis_exporter['listen_address'] = '0.0.0.0:9121' postgres_exporter['listen_address'] = '0.0.0.0:9187' gitaly['prometheus_listen_addr'] = "0.0.0.0:9236" gitlab_workhorse['prometheus_listen_addr'] = "0.0.0.0:9229" 

如有必要，请使用[官方安装说明](https://s0prometheus0io.icopy.site/docs/prometheus/latest/installation/)安装并设置专用的 Prometheus 实例.

\3. **将 Prometheus 服务器 IP 地址添加到**[**监视 IP 白名单**](https://www.kancloud.cn/apachecn/ip_whitelist.html) **. 例如：**

Plaintextgitlab_rails['monitoring_whitelist'] = ['127.0.0.0/8', '192.168.0.1'] 

\4. **在所有 GitLab Rails（Puma / Unicorn，Sidekiq）服务器上，设置 Prometheus 服务器 IP 地址和监听端口. 例如：**

Plaintextgitlab_rails['prometheus_address'] = '192.168.0.1:9090' 

\5. **要抓取 NGINX 指标，您还需要配置 NGINX 以允许 Prometheus 服务器 IP. 例如：**

Plaintextnginx['status']['options'] = {       "server_tokens" => "off",       "access_log" => "off",       "allow" => "192.168.0.1",       "deny" => "all", } 

\6. **重新配置 GitLab****以应用更改.**

Gogitlab-ctl reconfigure

\7. **编辑 Prometheus 服务器的配置文件.**

Govim /data/promethrus/prometheus.yml

\8. **将每个节点的导出器添加到 Prometheus 服务器的**[**抓取目标配置中**](https://s0prometheus0io.icopy.site/docs/prometheus/latest/configuration/configuration/) **. 例如，使用static_configs的样本片段：**

Plaintext``` scrape_configs:   - job_name: nginx     static_configs:       - targets:         - 1.1.1.1:8060   - job_name: redis     static_configs:       - targets:         - 1.1.1.1:9121   - job_name: postgres     static_configs:       - targets:         - 1.1.1.1:9187   - job_name: node     static_configs:       - targets:         - 1.1.1.1:9100   - job_name: gitlab-workhorse     static_configs:       - targets:         - 1.1.1.1:9229   - job_name: gitlab-rails     metrics_path: "/-/metrics"     static_configs:       - targets:         - 1.1.1.1:8080   - job_name: gitlab-sidekiq     static_configs:       - targets:         - 1.1.1.1:8082   - job_name: gitlab_exporter_database     metrics_path: "/database"     static_configs:       - targets:         - 1.1.1.1:9168   - job_name: gitlab_exporter_sidekiq     metrics_path: "/sidekiq"     static_configs:       - targets:         - 1.1.1.1:9168   - job_name: gitlab_exporter_process     metrics_path: "/process"     static_configs:       - targets:         - 1.1.1.1:9168   - job_name: gitaly     static_configs:       - targets:         - 1.1.1.1:9236  ```

\9. **重新加载 Prometheus 服务器.**

Gocurl -X POST http://localhost:9090/-/reload

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278640865-7bab0f27-865a-4ca6-a084-c18af6664ca5.png)

**使用prometheus监控harbor**

\1. **Harbor启用metrics服务**

1.1 停止Harbor服务

Plaintext$ cd /app/harbor               $ docker-compose  down

1.2 修改harbor.yml配置

修改harbor的配置文件中metrics参数，启用harbor-exporter
组件。

Plaintext$ cat harbor.yml ### metrics配置部分 metric:   enabled: true     #是否启用，需要修改为true（启用）   port: 9099       #默认的端口为9090，与prometheus的端口会冲突（所以需要修改下）   path: /metrics

对harbor不熟悉的建议对配置文件备份下！

1.3 配置注入组件

Plaintext$ ./prepre

1.4 install安装harbor

Plaintext$ ./install.sh  --with-notary  --with-trivy  --with-chartmuseum $ docker-compose ps NAME          COMMAND             SERVICE       STATUS            PORTS chartmuseum     "./docker-entrypoint…"   chartmuseum    running (healthy)    harbor-core     "/harbor/entrypoint.…"   core         running (healthy)    harbor-db      "/docker-entrypoint.…"   postgresql     running (healthy)   harbor-exporter  "/harbor/entrypoint.…"   exporter      running             

可以看到多了harbor-exporter组件。

\2.  **修改prometheus配置文件**

在prometheus的配置文件中指定获取harbor采集的指标数据。

Go#监控harbor   - job_name: "harbor-exporter"     scrape_interval: 20s     static_configs:       - targets: ['10.100.14.5:9099']         labels:           name: "harbor-exporter"   - job_name: 'harbor-core'     params:       comp: ['core']     static_configs:       - targets: ['10.100.14.5:9099']         labels:           name: "harbor-core"   - job_name: 'harbor-registry'     params:       comp: ['registry']     static_configs:       - targets: ['10.100.14.5:9099']         labels:           name: "harbor-registry"   - job_name: 'harbor-jobservice'     params:       comp: ['jobservice']     static_configs:       - targets: ['10.100.14.5:9099']         labels:           name: "harbor-jobservice"

\3. **重新加载 Prometheus 服务器.**

Gocurl -X POST http://localhost:9090/-/reload

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278641451-5373a4a1-a247-40ad-ae3d-721cce726ced.png)

\4. **配置告警规则**

Govim ./rules/node_exporter.yml  #以下暂时别加，测试有异常    - alert: NodeDiskUsage      expr: 100 - node_filesystem_free_bytes{fstype=~"xfs|ext4"} / node_filesystem_size_bytes{fstype=~"xfs|ext4"} * 100 > 80      for: 1m      labels:        severity: Warning      annotations:        summary: "{{$labels.instance}}: 分区使用率过高"        description: "{{$labels.instance}}: 分区使用大于 80% (当前值: {{ $value }}"    - alert: TCP连接数      expr: node_netstat_Tcp_CurrEstab > 1000      for: 1m      labels:        severity: Warning      annotations:        summary: "{{$labels.instance}}: TCP连接过高"        description: "{{$labels.instance}}: 连接大于1000 (当前值: {{$value}})"        - alert: IO      expr: 100 - (avg(irate(node_disk_io_time_seconds_total[1m])) by(instance)* 100) < 60      for: 1m      labels:        status: Warning      annotations:        summary: "{{$labels.instance}}: 流入磁盘IO使用率过高"        description: "{{$labels.instance}}:流入磁盘IO大于60%  (当前值:{{$value}})"

\5. **配置grafana仪表板（导入json模板）**

配置了Prometheus服务器以收集Harbor的指标，就可以使用 Grafana来可视化查看数据。Harbor 存储库中提供了一个 示例 Grafana 仪表板，可开始可视化 Harbor 指标。

下载地址：

https://github.com/goharbor/harbor/blob/main/contrib/grafana-dashborad/metrics-example.json

下载后打开grafana仪表板，依次点击：Import----->>>Upload JSON file----->>>选中刚刚下载的json文件----->>>打开选择数据源即可

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278642581-26528d16-bddb-4b44-bd33-b798b857b5cf.png)

**使用prometheus监控jenkins**

**1、Jenkins安装prometheus插件**

在Jenkins上，打开系统管理,插件管理，搜索Prometheus metrics 插件，勾选安装

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278643487-6eb7dea0-74c7-4e2d-bb8e-38e314f55f1c.png)

**2、Jenkins配置prometheus插件**

完成安装后，Jenkins重启，使安装的插件生效，然后进入系统管理下的系统设置

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278644118-8a14ea6b-6a7b-4204-81c5-d26930d5ac01.png)

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278644723-0bc706dd-c2c9-4759-873b-62371eb43c21.png)

设置完成后重启Jenkins服务，从url中查看指标情况

http://10.100.14.7:8080/jenkins-prometheus/

浏览器访问url就可以看到如下的效果

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278645307-3011f938-42d2-44ec-982f-246495d08a2e.png)

**3、Prometheus配置接入Jenkins的数据**

去prometheus服务器上修改prometheus.yml文件，追加

Govim prometheus.yml  - job_name: "jenkins"   metrics_path: "/jenkins-prometheus/"   scheme: http   static_configs:   - targets: ['10.100.14.7:8080']

重新加载 Prometheus 服务器配置.

Gocurl -X POST http://localhost:9090/-/reload

**4、Grafana配置来自Prometheus的数据可视化**

在grafana中导入图表模板，模板的id是9964

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278645906-9c9e5e87-de94-4b15-9cef-b038e8162556.png)

**使用prometheus监控nexus**

**metrics 采集**

Nexus 提供了 Prometheus 格式数据的接口，这个接口需要授权 nx-metrics-all 才能访问，所以，将 metrics 采集到 Prometheus 里，需要做如下。

metrics 接口：http://10.100.14.10:8081/service/metrics/prometheus

**创建 metrics 接口权限账户**

给指标接口创建一个单独的 metrics 账户，然后创建一个新的 Role（nx-metrics） , 将 nx-metrics-all 给这个角色，然后将角色给 metrics 账户。如下图。

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278646906-129579af-b196-4aa5-be50-dec8b36be4bb.png)

**配置 Prometheus** 

metrics 账户准备好后，在 Prometheus 侧添加如下配置项：

Goglobal:   scrape_interval: 15s   scrape_timeout: 10s   evaluation_interval: 15s alerting:   alertmanagers:   - static_configs:     - targets: []     scheme: http     timeout: 10s scrape_configs: - job_name: nxrm   scrape_interval: 15s   scrape_timeout: 10s   metrics_path: /service/metrics/prometheus   scheme: http   basic_auth:     username: metrics     password: metrics   static_configs:   - targets:     - 10.100.14.10:8081

配置无误后，metrics 已经收集到 Prometheus 了。可以在 Prometheus 里查询  org_sonatype_nexus_ 打头的指标，有就代表配置成功了。

**grafana 绘制面板（ID：16459）**

Nexus metrics 总共有超过 2k 多行。在绘制观测面板前，需要先分析下采集的 metrics 信息

**分析 metrics** 

Nexus 暴露出来的指标，主要分三个类别：

Component ：相关组件的运行时指标，比如文件存储组件 FileBlobStore，以  org_sonatype_nexus_ 前缀开头的系列指标。

Jetty ：Web 容器，负责接收响应请求的，以  org_eclipse_jetty_ 前缀开头的系列指标。

Jvm ：Jvm Runtime，反应堆内存、非堆内存、GC 回收等，以 jvm_ 前缀开头的系列指标。

其中，Component 指标是最多的，也有部分 Component 根本就没有使用，指标一直为 0，这部分指标就可以不用绘制面板。

**绘制观测面板**

经过上面分析，可以绘制出如下观测面板。

**Nexus Component**

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278648787-f0d32e1f-4753-42b6-a51f-d0574ffe8459.png)

这个面板绘制了常用的组件运行信息，以 LdapServerComponent 为例，我们启用了这个组件，对接了公司内部的 LDAP 。之前有用户反馈登录失败，最后排查到是 Nexus 访问 LDAP Server 出现了问题，如果有了这个观测图，就能很清楚的看到 LdapServerComponent 内部的运行情况，快速定位问题。另外，Component Exceptions 面板也能集中观测到所有组件的异常信息，这里只要指标大于 0 ，就代表 Nexus 服务出问题了。

**Jetty WebAppContext**

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278649968-b67a0b5c-f213-4c30-a67f-1553b6257792.png)

Jetty 面板反应了 Nexus 的流量情况，请求响应等信息。其中 Jetty 线程池，可以关注 wait-jobs（等待线程池执行的任务数） 这个指标，如果这个指标一直大于 0 ，则表示 Nexus Server 出现性能问题了。

**Jvm Runtime**

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278650600-ba65c9e5-e03a-4842-a22c-8cefdb844ee7.png)

Jvm 的指标不用多说，和一般应用的 jvm 一样。它反应了应用的内存情况，包括堆内存和非堆内存，以及堆外内存，还有 GC 执行情况等。以内存观测为例，我们的 Nexus 服务是从 ECS 迁移的 K8s 环境中的，当时给 pod 的内存资源设置了 request 、limit 4G，给 jvm xmx 设置了 2G，运行后还会出现内存告警，实际使用内存远超 xmx 设置的大小。从贴出截图已经可以看出答案，是因为 Nexus 使用了超过 2G 的堆外内存（Direct Buffers + Mapped Buffers）。

**使用prometheus监控nacos**

\1. **修改nacos配置**

首先进入nacos容器添加一行配置

配置文件路径在/usr/local/nacos/conf/application.properties

Govim /usr/local/nacos/conf/application.properties #新增一行配置  暴露metrics数据 management.endpoints.web.exposure.include=*

如果是集群nacos每个节点都添加即可

配置完成后重启nacos

随后访问prometheus采集的数据看是否成功

http://10.100.14.19:8848/nacos/actuator/prometheus

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278651081-444a4a8b-2d2a-42ab-80b7-2e01c5aef177.png)

访问到内容则表示采集成功

\2. **配置prometheus的配置文件**

Govim prometheus.yml #添加以下配置   - job_name: 'nacos'     metrics_path: '/nacos/actuator/prometheus'     static_configs:       - targets:         - 10.100.14.19:8848         labels:           name: "nacos19"       - targets:         - 10.100.14.20:8848         labels:           name: "nacos20"       - targets:         - 10.100.14.21:8848         labels:           name: "nacos21" #注意缩进问题和上面对齐不然会报错！！！

\3. **重新加载 Prometheus 配置文件**

Gocurl -X POST http://localhost:9090/-/reload

访问prometheus  web页面 http://10.100.14.23:9090/targets?search=nacos

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278652036-2f79fa97-b92f-426c-a4bb-bdbfb1da728b.png)

\4. **配置grafana（仪表盘获取不到数据13221）**

**使用prometheus监控数据库MySQL**

\1. **安装mysql_exporter**

Gotar -zxvf mysqld_exporter-0.14.0.linux-amd64.tar.gz mv mysqld_exporter-0.14.0.linux-amd64.tar.gz mysqld_exporter  #在mysqld-exporter安装路径下，创建.my.cnf文件。内容如下: [client] user=mysql_monitor password=Mysql@123  #创建mysql 用户并授权 CREATE USER 'mysql_monitor'@'localhost' IDENTIFIED BY 'Mysql@123' WITH MAX_USER_CONNECTIONS 3; GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysql_monitor'@'localhost'; FLUSH PRIVILEGES; EXIT  **#添加system系统服务**vim /usr/lib/systemd/system/mysqld_exporter.service [Unit] Description=mysqld_exporter After=network.target  [Service] User=root Type=simple ExecStart=/data/mysqld_exporter/mysqld_exporter --config.my-cnf /data/mysqld_exporter/.my.cnf --collect.info_schema.processlist  Restart=on-failure  [Install] WantedBy=multi-user.target   **#启动添加的system服务**systemctl daemon-reload systemctl start mysqld_exporter.service netstat -lntup | grep "9104"

**查看mysql 监控信息**
浏览器运行 http://10.100.14.42:9104/metrics，查看是否metrics数据输出，如果有输出内容监控就正常

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278652674-29f91b51-dfaf-4b5f-a567-7b940a34bf42.png)

\2. **配置prometheus**

Govim /data/prometheus/prometheus.yml  监控MySQL   - job_name: 'mysqld_agent'   # 单机版     static_configs:       - targets: ["10.100.14.27:9104"]         labels:           name: "mysql-dev"   - job_name: 'mysql-mater-slave'  #主从集群     scrape_interval: 5s     static_configs:       - targets: ["10.100.14.42:9104"]         labels:           name: "mysql-prod42"       - targets: ["10.100.14.43:9104"]         labels:           name: "mysql-prod43"  #重新加载 Prometheus 配置文件curl -X POST http://localhost:9090/-/reload

\3. **配置grafana**

Go下载模板 https://grafana.com/grafana/dashboards/7371  ## 主从看板 https://grafana.com/grafana/dashboards/7362  ## 主看板 https://grafana.com/grafana/dashboards/7365  ## innodb看板 在grafana中导入dashboard

\4. **配置告警**

在prometheus目录下编辑报警模版mysql_rules.yml，添加一些自定义报警项。

Govim /data/prometheus/rules/mysql_rules.yml groups: - name: MySQL-rules   rules:   - alert: MySQL Status      expr: up == 0for: 5s      labels:       severity: warning     annotations:       summary: "{{$labels.instance}}: MySQL has stop !!!"       description: "检测MySQL数据库运行状态"    - alert: MySQL Slave IO Thread Status     expr: mysql_slave_status_slave_io_running == 0for: 5s      labels:       severity: warning     annotations:        summary: "{{$labels.instance}}: MySQL Slave IO Thread has stop !!!"       description: "检测MySQL主从IO线程运行状态"    - alert: MySQL Slave SQL Thread Status      expr: mysql_slave_status_slave_sql_running == 0for: 5s      labels:       severity: warning     annotations:        summary: "{{$labels.instance}}: MySQL Slave SQL Thread has stop !!!"       description: "检测MySQL主从SQL线程运行状态"    - alert: MySQL Slave Delay Status      expr: mysql_slave_status_sql_delay == 30for: 5s      labels:       severity: warning     annotations:        summary: "{{$labels.instance}}: MySQL Slave Delay has more than 30s !!!"       description: "检测MySQL主从延时状态"     - alert: Mysql_Too_Many_Connections     expr: rate(mysql_global_status_threads_connected[5m]) > 200for: 2m     labels:       severity: warning     annotations:       summary: "{{$labels.instance}}: 连接数过多"       description: "{{$labels.instance}}: 连接数过多，请处理 ,(current value is: {{ $value }})"       - alert: Mysql_Too_Many_slow_queries     expr: rate(mysql_global_status_slow_queries[5m]) > 3for: 2m     labels:       severity: warning     annotations:       summary: "{{$labels.instance}}: 慢查询有点多，请检查处理"       description: "{{$labels.instance}}: Mysql slow_queries is more than 3 per second ,(current value is: {{ $value }})"

**使用prometheus监控数据库PostgreSQL**

\1. **安装postgres_exporter**

Gotar zxvf postgres_exporter-0.11.1.linux-amd64.tar.gz mv postgres_exporter-0.11.1.linux-amd64.tar.gz postgres_exporter  **#添加system系统服务** vim /etc/systemd/system/postgres_exporter.service   [Unit] Description=postgres_exporter After=network.target [Service] Type=simple User=root Environment="DATA_SOURCE_NAME=postgresql://postgres:postgres@127.0.0.1:5432/postgres?sslmode=disable"   #注意postgres密码 ExecStart=/data/postgres_exporter/postgres_exporter Restart=on-failure [Install] WantedBy=multi-user.target  systemctl daemon-reload systemctl start postgres_exporter.service netstat -lntup | grep "9187"

**查看postgres 监控信息**
浏览器运行 http://10.100.14.46:9187/metrics，查看是否metrics数据输出，如果有输出内容监控就正常

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278653703-a0138cd6-77b5-49f8-a018-4c6c9c88a5a0.png)

\2. **配置prometheus**

Govim /data/prometheus/prometheus.yml  监控postgresql   - job_name: 'postgres'     static_configs:     - targets: ['10.100.14.28:9187']       labels:         name: "postgres-dev"     - targets: ['10.100.14.46:9187']       labels:         name: "postgres-pro"  #重新加载 Prometheus 配置文件curl -X POST http://localhost:9090/-/reload

\3. **配置grafana**

grafanaid: 455

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278655137-a85ef545-a0c9-475a-b487-907c2bd4594c.png)

grafanaid: 9628 

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278656167-10bfddd2-32d9-42c7-9d18-d6a02b477d65.png)

\4. **配置告警**

Govim /data/prometheus/rules/postgres.rules  groups: - name: postgresql-监控告警       rules:   - alert: 警报！Postgresql宕机     expr: pg_up == 0     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql down"       description: "Postgresql instance is down\n  当前值={{ $value }}"    - alert: 警报！Postgresql被重启     expr: time() - pg_postmaster_start_time_seconds < 60     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql restarted"       description: "Postgresql restarted\n  当前值={{ $value }}"    - alert: 警报！PostgresqlExporterError     expr: pg_exporter_last_scrape_error > 0     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql exporter error"       description: "Postgresql exporter is showing errors. A query may be buggy in query.yaml\n  当前值={{ $value }}"    - alert: 警报！Postgresql主从复制不同步     expr: pg_replication_lag > 30 and ON(instance) pg_replication_is_replica == 1     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql replication lag"       description: "PostgreSQL replication lag is going up (> 30s)\n  当前值={{ $value }}"    - alert: 警报！PostgresqlTableNotVaccumed     expr: time() - pg_stat_user_tables_last_autovacuum > 60 * 60 * 24     for: 0m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql table not vaccumed"       description: "Table has not been vaccum for 24 hours\n  当前值={{ $value }}"    - alert: 警报！PostgresqlTableNotAnalyzed     expr: time() - pg_stat_user_tables_last_autoanalyze > 60 * 60 * 24     for: 0m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql table not analyzed"       description: "Table has not been analyzed for 24 hours\n  当前值={{ $value }}"    - alert: 警报！Postgresql连接数太多     expr: sum by (datname) (pg_stat_activity_count{datname!~"template.*|postgres"}) > pg_settings_max_connections * 0.8     for: 2m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql too many connections"       description: "PostgreSQL instance has too many connections (> 80%).\n  当前值={{ $value }}"    - alert: 警报！Postgresql连接数太少     expr: sum by (datname) (pg_stat_activity_count{datname!~"template.*|postgres"}) < 5     for: 2m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql not enough connections"       description: "PostgreSQL instance should have more connections (> 5)\n  当前值={{ $value }}"    - alert: 警报！Postgresql死锁     expr: increase(pg_stat_database_deadlocks{datname!~"template.*|postgres"}[1m]) > 5     for: 0m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql dead locks"       description: "PostgreSQL has dead-locks\n  当前值={{ $value }}"    - alert: 警报！Postgresql慢查询     expr: pg_slow_queries > 0     for: 2m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql slow queries"       description: "PostgreSQL executes slow queries\n  当前值={{ $value }}"    - alert: 警报！Postgresql回滚率高     expr: rate(pg_stat_database_xact_rollback{datname!~"template.*"}[3m]) / rate(pg_stat_database_xact_commit{datname!~"template.*"}[3m]) > 0.02     for: 0m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql high rollback rate"       description: "Ratio of transactions being aborted compared to committed is > 2 %\n  当前值={{ $value }}"    - alert: 警报！Postgresql提交率低     expr: rate(pg_stat_database_xact_commit[1m]) < 10     for: 2m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql commit rate low"       description: "Postgres seems to be processing very few transactions\n  当前值={{ $value }}"    - alert: 警报！PostgresqlLowXidConsumption     expr: rate(pg_txid_current[1m]) < 5     for: 2m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql low XID consumption"       description: "Postgresql seems to be consuming transaction IDs very slowly\n  当前值={{ $value }}"    - alert: 警报！PostgresqllowXlogConsumption     expr: rate(pg_xlog_position_bytes[1m]) < 100     for: 2m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresqllow XLOG consumption"       description: "Postgres seems to be consuming XLOG very slowly\n  当前值={{ $value }}"    - alert: 警报！PostgresqlWaleReplicationStopped     expr: rate(pg_xlog_position_bytes[1m]) == 0     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql WALE replication stopped"       description: "WAL-E replication seems to be stopped\n  当前值={{ $value }}"    - alert: 警报！PostgresqlHighRateStatementTimeout     expr: rate(postgresql_errors_total{type="statement_timeout"}[1m]) > 3     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql high rate statement timeout"       description: "Postgres transactions showing high rate of statement timeouts\n  当前值={{ $value }}"    - alert: 警报！PostgresqlHighRateDeadlock     expr: increase(postgresql_errors_total{type="deadlock_detected"}[1m]) > 1     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql high rate deadlock"       description: "Postgres detected deadlocks\n  当前值={{ $value }}"    - alert: 警报！PostgresqlReplicationLagBytes     expr: (pg_xlog_position_bytes and pg_replication_is_replica == 0) - on(environment) group_right(instance) (pg_xlog_position_bytes and pg_replication_is_replica == 1) > 1e+09     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql replication lag bytes"       description: "Postgres Replication lag (in bytes) is high\n  当前值={{ $value }}"    - alert: 警报！PostgresqlUnusedReplicationSlot     expr: pg_replication_slots_active == 0     for: 1m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql unused replication slot"       description: "Unused Replication Slots\n  当前值={{ $value }}"    - alert: 警报！PostgresqlTooManyDeadTuples     expr: ((pg_stat_user_tables_n_dead_tup > 10000) / (pg_stat_user_tables_n_live_tup + pg_stat_user_tables_n_dead_tup)) >= 0.1 unless ON(instance) (pg_replication_is_replica == 1)     for: 2m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql too many dead tuples"       description: "PostgreSQL dead tuples is too large\n  当前值={{ $value }}"    - alert: 警报！PostgresqlSplitBrain     expr: count(pg_replication_is_replica == 0) != 1     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql split brain"       description: "Split Brain, too many primary Postgresql databases in read-write mode\n  当前值={{ $value }}"    - alert: 警报！PostgresqlPromotedNode     expr: pg_replication_is_replica and changes(pg_replication_is_replica[1m]) > 0     for: 0m     labels:       severity: 一般告警     annotations:       summary: "{{$labels.instance}} Postgresql promoted node"       description: "Postgresql standby server has been promoted as primary node\n  当前值={{ $value }}"    - alert: 警报！PostgresqlSslCompressionActive     expr: sum(pg_stat_ssl_compression) > 0     for: 0m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql SSL compression active"       description: "Database connections with SSL compression enabled. This may add significant jitter in replication delay. Replicas should turn off SSL compression via `sslcompression=0` in `recovery.conf`.\n  当前值={{ $value }}"    - alert: 警报！PostgresqlTooManyLocksAcquired     expr: ((sum (pg_locks_count)) / (pg_settings_max_locks_per_transaction * pg_settings_max_connections)) > 0.20     for: 2m     labels:       severity: 严重告警     annotations:       summary: "{{$labels.instance}} Postgresql too many locks acquired"       description: "Too many locks acquired on the database. If this alert happens frequently, we may need to increase the postgres setting max_locks_per_transaction.\n  当前值={{ $value }}"

**使用prometheus监控数据库MongoDB**

\1. **安装mongodb_exporter**

Gotar -zxvf mongodb_exporter-0.35.0.linux-amd64.tar.gz  mv mongodb_exporter-0.35.0.linux-amd64 mongodb_exporter  **#添加system系统服务** vim /usr/lib/systemd/system/mongodb_exporter.service  [Unit] Description=mongodb_exporter Documentation=https://github.com/percona/mongodb_exporter After=network.target [Service] Type=simple User=root Environment="MONGODB_URI=mongodb://root:123321@localhost:30000" ExecStart=/data/mongodb_exporter/mongodb_exporter --log.level=error  Restart=on-failure [Install] WantedBy=multi-user.target  systemctl daemon-reload  systemctl restart mongodb_exporter.service  netstat -nltp |grep 9216

**查看postgres 监控信息**
浏览器运行 http://10.100.14.34:9216/metrics，查看是否metrics数据输出，如果有输出内容监控就正常

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278656641-48b188bf-41e8-4439-a5b7-58670d578fc7.png)

\2. **配置prometheus**

Govim /data/prometheus/prometheus.yml  监控MongoDB   - job_name: 'mongodb'     static_configs:     - targets: ['10.100.14.34:9216']       labels:         name: "mongos"         project: mongo         unitname: "Mongodb_exporter"         service: mongo #重新加载 Prometheus 配置文件curl -X POST http://localhost:9090/-/reload

\3. **配置grafana**

2583

16974

暂无数据展示

**使用prometheus监控数据库Redis**

\1. **安装redis_exporter**

Gotar zxvf redis_exporter-v1.45.0.linux-amd64.tar.gz mv redis_exporter-v1.45.0.linux-amd64 redis_exporter  **#添加system系统服务**vim /usr/lib/systemd/system/redis_exporter.service  [Unit] Description=redis_exporter Documentation=https://github.com/oliver006/redis_exporter After=network.target [Service] Type=simple User=prometheus ExecStart=/data/redis_exporter/redis_exporter -redis.addr 127.0.0.1:6379  -redis.password vHLjMgF%knQ@Ba%3 Restart=on-failure [Install] WantedBy=multi-user.target     **#启动添加的system服务**systemctl daemon-reload systemctl start redis_exporter.service netstat -lntup | grep "9121"

**查看mysql 监控信息**
浏览器运行 http://10.100.14.49:9121/metrics，查看是否metrics数据输出，如果有输出内容监控就正常

![img](assets/部署prometheus+grafana+alertmanager+钉钉监控中心/1669278657844-cbef9a03-1722-45b7-a56e-93d3b27b2f8b.png)

\2. **配置prometheus**

Govim /data/prometheus/prometheus.yml  监控Redis   - job_name: 'redis'     scrape_interval: 10s     static_configs:       - targets: ['10.100.14.49:9121']         labels:           instance: redis49           name: redis49       - targets: ['10.100.14.50:9121']         labels:           instance: redis50           name: redis50       - targets: ['10.100.14.51:9121']         labels:           instance: redis51           name: redis51   #重新加载 Prometheus 配置文件curl -X POST http://localhost:9090/-/reload

\3. **配置grafana**

Go下载模板  在grafana中导入dashboard11835

\4. **配置告警**

在prometheus目录下编辑报警模版mysql_rules.yml，添加一些自定义报警项。

Govim /data/prometheus/rules/redis_rules.yml groups: - name: redis集群预警   rules:   - alert: "redis节点下线"     expr: up{instance=~".*:9121"} == 0     for: 20s     labels:       severity: ERROR       alert_type: "节点下线通知"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }} redis 监控主节点下线, 请及时处理 命令: systemctl restart|status redis-exporter"    - alert: "redis节点下线"     expr: redis_up{instance=~"redis:.*"} == 0     for: 20s     labels:       severity: WARN       alert_type: "节点下线通知"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "{{ $labels.instance }} 节点下线, 请及时处理"    - alert: "redis集群节点丢失"     expr: time() - redis_rdb_last_save_timestamp_seconds > 60 * 60 * 24     for: 5m     labels:       severity: ERROR     annotations:       summary: "Missing backup (instance {{ $labels.instance }})"       description: "redis 集群节点： {{$labels.instance}}  已下线24小时, 请立即处理"      - alert: "内存使用大于95%"     expr: redis_memory_used_bytes / redis_total_system_memory_bytes * 100 > 95     for: 5m     labels:       severity: WARN     annotations:       description: "Redis 当前节点 {{ $labels.instance }} 内存已使用 {{ $value }}%"    - alert: "内存异常"     expr: redis_mem_fragmentation_ratio < 1     for: 5m     labels:       severity: WARN     annotations:       description: "Redis 当前节点 {{ $labels.instance }} redis内存可用内存不足,请减少key或增加内存"    - alert: "内存异常"     expr: redis_mem_fragmentation_ratio > 18     for: 5m     labels:       severity: ERROR     annotations:       description: "Redis 当前节点 {{ $labels.instance }} 内存碎片过大, 当前: {{ $value}}, 处理"    - alert: "redis连接被拒绝"     expr: increase(redis_rejected_connections_total[1m]) > 0     for: 5m     labels:       severity: WARN       alert_type: "连接被拒绝"     annotations:       description: "redis 一些服务连接 {{ $labels.instance }} 被拒绝: 查看文档"    - alert: "redis主节点缺失"     expr: count(redis_instance_info{role="master"}) == 0     for: 5m     labels:       severity: WARN       alert_type: "redis主节点缺失"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       summary: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }} redis主节点缺失"       description: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }} 主节点丢失5分钟"    - alert: "redis副本下线"     expr: delta(redis_connected_slaves[1m]) < 0     for: 5m     labels:       severity: WARN       alert_type: "数据不同步"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "redis {{ $labels.instance}} 集群副本下线, 请立即处理"      - alert: "redis连接数过多"     expr: redis_connected_clients{instance=~"redis://.*"} > redis_config_maxclients{instance=~"redis://.*"} * 0.85     for: 5m     labels:       severity: WARN       alert_type: "连接数过多"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "主机: {{ $labels.instance }} 当前连接数： {{ $value }}， 连接总数达到总量的85%，请立即检查"    - alert: "redis连接数过多"     expr: redis_connected_clients{instance=~"redis://.*"} > redis_config_maxclients{instance=~"redis://.*"} * 0.95     for: 5m     labels:       severity: ERROR       alert_type: "连接数过多"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "主机: {{ $labels.instance }} 当前连接数： {{ $value }}， 连接总数达到总量的95%，请立即检查"    - alert: "redis 连接数过低"     expr: redis_connected_clients == 0     for: 5m     labels:       severity: WARN       alert_type: "连接数过低"       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "当前: {{ $labels.instance }} 无连接， 请检查下服务是否下线了"    - alert: "redis连接故障"     expr: irate(redis_blocked_clients{job="redis_exporter_targets"}[5m]) > 3     for: 5m     labels:       severity: WARN       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "当前: {{ $labels.alert_host }} 5分钟内阻塞进程大于 3, 请检查连接服务是否异常"    - alert: "redis低命中率效率低下"     expr: redis_keyspace_hits_total / (redis_keyspace_hits_total + redis_keyspace_misses_total) > 0.95     for: 5m     labels:       severity: ERROR       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "当前: {{ $labels.alert_host }} 命中率低下原因： 数据到期和分配给Redis的内存不足，请及时检查内存、数据"    - alert: "redis异常同步"     expr: irate(redis_rdb_changes_since_last_save[60m])     for: 60m     labels:       severity: ERROR       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "当前: {{ $labels.alert_host }} redis 某一台服务异常断开, 同步异常"    - alert: "redis集群连接异常"     expr: redis_master_link_up{master_host=~".*"} == 0     for: 5m     labels:       severity: WARN       alert_host: "{{ reReplaceAll \":(.*)\" \"\" $labels.instance }}"     annotations:       description: "当前: {{ $labels.alert_host }} redis 复制连接当前断开"

**使用prometheus监控站点**

主要用来监控部署的中间件管理页面，如：禅道、sentinel、定时任务xxl-job等，还有公司的线上项目前后台等。

\1. **接入ConsulManager：**

登陆ConsulManager控制台，依次点击站点与接口监控----->>>站点管理----->>>导出：

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278659270-ed990b9e-b608-4c91-aa73-efcb8841492b.png)

编辑导出的模板（根据实际情况填写），保存：

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278660511-49e3646f-ae07-490c-a3df-7b7896f67f70.png)

回到ConsulManager控制台的站点管理，点击导入，选择刚刚编辑的文件，打开之后就可以看到所有需要监控的站点了：

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278660924-7745a376-acac-444b-b6ec-0f33620dbbd5.png)

\2. **接入blackbox**

接下来配置blackbox，点击blackbox配置，可以看到ConsulManager已经提供了完善的模板，直接复制去监控服务器粘贴即可：

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278662090-99a3cb08-685c-4d1c-a80a-793e5ea3a4e2.png)

Govim /data/blackbox_exporter/blackbox.yml

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278663573-a8676d43-391e-46f6-b543-7bbb17354820.png)

\3. **接入prometheus**

继续复制prometheus配置：

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278664176-27b3411f-880f-4aa8-b862-2802342aa692.png)

Govim /data/prometheus/prometheus.yml

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278665070-cd024233-26c1-4cbd-b25b-8cc06f3a76ab.png)

\4. **配置告警**

复制告警规则：

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278665637-191cd752-194a-4d83-bbd0-c80ec73ee74d.png)

Go vim /data/prometheus/rules/http.yml 

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278666258-0471dea8-f281-4b1d-8f38-b64d5545e920.png)

编辑完以后热加载prometheus配置文件

Gocurl -X POST http://localhost:9090/-/reload

\5. **接入grafana（9965）**

![img](https://cdn.nlark.com/yuque/0/2022/png/23214851/1669278667098-37ddcde7-7ca6-4e4e-be39-175e3bae0bdc.png)