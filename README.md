#�رշ���ǽ��selinux
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i '/SELINUX/s/enforcing/disabled/' /etc/selinux/config
timedatectl set-timezone Asia/Shanghai

#����glusterfs�洢�ڵ㰲װglusterfs
yum -y install centos-release-gluster
yum -y install glusterfs-server
systemctl enable glusterd
systemctl start glusterd

#K8S�ڵ���Ҫ��װglusterfs�ͻ���
yum -y install centos-release-gluster
yum -y install glusterfs-client
systemctl enable glusterfs-client
systemctl start glusterfs-client


#����hosts
cat >> /etc/hosts << EOF
192.168.10.4 docker.masters
192.168.10.5 docker1.node
192.168.10.3 node2
EOF


#��ĳ̨glusterfs�ڵ���ִ��
yum -y install heketi heketi-client
ssh-keygen -t rsa


cat ~/.ssh/id_rsa.pub | ssh 192.168.10.5 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh 1192.168.10.3 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"


#����heketi.json
tee  > /etc/heketi/heketi.json<<EOF
{
  "_port_comment": "Heketi Server Port Number",
  "port": "8081",

  "_use_auth": "Enable JWT authorization. Please enable for deployment",
  "use_auth": false,

  "_jwt": "Private keys for access",
  "jwt": {
    "_admin": "Admin has access to all APIs",
    "admin": {
      "key": "My Secret"
    },
    "_user": "User only has access to /volumes endpoint",
    "user": {
      "key": "My Secret"
    }
  },

  "_glusterfs_comment": "GlusterFS Configuration",
  "glusterfs": {
    "_executor_comment": [
      "Execute plugin. Possible choices: mock, ssh",
      "mock: This setting is used for testing and development.",
      "      It will not send commands to any node.",
      "ssh:  This setting will notify Heketi to ssh to the nodes.",
      "      It will need the values in sshexec to be configured.",
      "kubernetes: Communicate with GlusterFS containers over",
      "            Kubernetes exec api."
    ],
    "executor": "ssh",

    "_sshexec_comment": "SSH username and private key file information",
    "sshexec": {
      "keyfile": "/root/.ssh/id_rsa",
      "user": "root",
      "port": "22",
      "fstab": "/etc/fstab"
    },

    "_kubeexec_comment": "Kubernetes configuration",
    "kubeexec": {
      "host" :"https://kubernetes.host:8443",
      "cert" : "/path/to/crt.file",
      "insecure": false,
      "user": "kubernetes username",
      "password": "password for kubernetes user",
      "namespace": "OpenShift project or Kubernetes namespace",
      "fstab": "Optional: Specify fstab file on node.  Default is /etc/fstab"
    },

    "_db_comment": "Database file name",
    "db": "/var/lib/heketi/heketi.db",

    "_loglevel_comment": [
      "Set log level. Choices are:",
      "  none, critical, error, warning, info, debug",
      "Default is warning"
    ],
    "loglevel" : "debug"
  }
}
EOF

systemctl enable heketi 
systemctl restart heketi


#���ڿ�����������ʧ�������޸��ļ�
tee >/usr/lib/systemd/system/heketi.service <<EOF
[Unit]
Description=Heketi Server

[Service]
Type=simple
WorkingDirectory=/var/lib/heketi
EnvironmentFile=/etc/heketi/heketi.json
User=root
ExecStart=/usr/bin/heketi --config=/etc/heketi/heketi.json
Restart=on-failure
StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl restart heketi
curl http://localhost:8081/hello

#����heketi������Ϣ
export HEKETI_CLI_SERVER=http://localhost:8081  
tee >/etc/heketi/topology.json <<EOF
{
  "clusters": [
    {
      "nodes": [
        {
          "node": {
            "hostnames": {
              "manage": [
                "192.168.10.3"
              ],
              "storage": [
                "192.168.10.3"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "192.168.10.4"
              ],
              "storage": [
                "192.168.10.4"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        },
        {
          "node": {
            "hostnames": {
              "manage": [
                "192.168.10.5"
              ],
              "storage": [
                "192.168.10.5"
              ]
            },
            "zone": 1
          },
          "devices": [
            "/dev/sdb"
          ]
        }
      ]
    }
  ]
}
EOF
#������Ⱥ
heketi-cli topology load --json=/etc/heketi/topology.json

#����heketi����
heketi-cli cluster list
heketi-cli node list
heketi-cli node info ID
heketi-cli topology info

ɾ��pv
heketi-cli volume list
heketi-cli volume delete 426d9bb6303265582ee19157c6d5abd8

�鿴ʣ��ռ��С
for i in `heketi-cli node list|awk '{print $1}'|awk -F ':' '{print$ 2}'`;do heketi-cli node info $i|grep Size;done


#������̬����
gluserfs-storage-class.yaml


kubectl create -f gluserfs-storage-class.yaml

#���ɲ���pod��ʹ��pvc��Ϣ
pod.yaml

kubectl create -f pod.yaml

#�鿴pod�Ĺ�����Ϣ

kubectl exec -it `kubectl get po |grep Running|awk '{print $1}'` -- df -h 
