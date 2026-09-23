**Key Components**
1. Access Management
Centralized Access: Ansible runs from a central Control Node using a Service Account. The account has only the permissions it needs: Compute Viewer to find VMs and IAP-secured Tunnel User to connect to them.
Secure SSH Connection using IAP: We don't use static SSH keys or expose VMs to the internet. Instead, Google Cloud IAP creates a temporary, encrypted tunnel between Ansible and the VM.
2. VM Discovery and Monitoring Setup
Automatic VM Discovery: We use the google.cloud.gcp_compute Ansible plugin to automatically discover VMs across all authorized GCP projects.
Automatic VM Enrollment: VMs are selected using a label such as monitor: true. When a new VM is created with this label, Ansible automatically finds it and installs/configures the Google Cloud Ops Agent during the next playbook run. No manual inventory update is required.
Centralized Monitoring: The Ops Agent collects disk usage metrics such as disk/percent_used. A Metrics Scope brings monitoring data from multiple GCP projects into one central Monitoring Project, allowing the enterprise to create common dashboards and alerts.

**GCP Dynamic Inventory Configuration**
plugin: google.cloud.gcp_compute
projects:

source-project-123

source-project-456
auth_kind: serviceaccount

**The service account needs Compute Viewer and IAP Tunnel User permissions**

service_account_file: /opt/ansible/sa-credentials.json
filters:

status = RUNNING

labels.monitor = true
hostnames:

name
compose:

**Dynamically route SSH traffic through Google Cloud IAP**

ansible_ssh_common_args: >
-o ProxyCommand='gcloud compute start-iap-tunnel %h %p --listen-on-stdin --project={{ project }} --zone={{ zone }}'

**Main Deployment Playbook**

- name: Deploy Google Cloud Ops Agent for Disk Monitoring
  hosts: all
  become: yes
  gather_facts: yes
  roles:
    - ops_agent
 
**Ops Agent Tasks**

#Agent Installation & Config 
name: Add Google Cloud Ops Agent repository and install script
shell: |
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
args:
creates: /etc/google-cloud-ops-agent/config.yaml
notify: Restart Ops Agent

name: Push Ops Agent custom metrics configuration
copy:
src: config.yaml
dest: /etc/google-cloud-ops-agent/config.yaml
owner: root
group: root
mode: '0644'
notify: Restart Ops Agent

**Ops Agent Handlers**
- name: Restart Ops Agent
  systemd:
    name: google-cloud-ops-agent
    state: restarted
    enabled: yes

**Ops Agent Configuration File**

This configuration specifically captures host metrics (including disk utilization) at a 60-second interval to ensure rapid detection of low disk space.

#capture the standard host metrics, which includes disk utilization, every 60 seconds.

metrics:
  receivers:
    hostmetrics:
      type: hostmetrics
      collection_interval: 60s
  service:
    pipelines:
      default_pipeline:
        receivers: [hostmetrics]

      
