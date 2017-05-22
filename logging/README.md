# Preface
Centos/RHEL 7 comes with services which saves logging information. Some services write their own logs directly to their log information files, e.g. apache maintain their own logs. Some of the service maintain their logs through systemctl. Systemctl is a services that take care of starting, stopping or monitoring the status of a process. systemctl further communicates to Journald which keep track on log information. “journalctl” is used to grep log inforamtion from journald.

# Definition of Journal
Journal is a component of systemd. It capture log messages of kernel logs, syslog messages, or error log messages. It collect them, index them and makes availabe to the users. Journal are stored in /run/log/journal directory.
```
# journaldctl -f
# journalctl -p err
# journalctl --disk-usage
# journalctl --since yesterday
# 
```
# Build Journald Logstash Docker image (one of following steps, you can use official images or build by yourself.)
## Build by yourself
```
# cd logging/docker
# docker build -f Dockerfile.self . -t machozhao/logstash-journald
```
## Build from Offical image
```
# cd logging/docker
# docker build -f Dockerfile.office . -t machozhao/logstash-journald
```

# Run Pod
## Start Pod
```
kubectl create -f deploy/kubernetes
```
## Acccess Elasticsearch and get indices
```
http://node1:31602/_cat/indices?v
```

# References
* https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
