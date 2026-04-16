
**OpenStack cannot run on the AWS Free Tier.** The AWS Free Tier provides `t2.micro` or `t3.micro` instances, which only have 1 vCPU and 1 GB of RAM. OpenStack (even the lightweight DevStack version) strictly requires a minimum of 4 CPU cores, 8 GB of RAM, and 50 GB of disk space. Furthermore, DevStack requires nested virtualization (a hypervisor running inside a hypervisor), which AWS only supports on their expensive `.metal` instances. If you attempt to run this setup on a Free Tier instance, the server will crash instantly. 

If you want to perform this on AWS, you will need to provision a larger instance (like a `t3.xlarge` or `m5.xlarge`) and accept that **it will incur hourly charges**. 

With that reality check out of the way, here is the consolidated master guide for setting up your OpenStack environment, tailored into a logical, start-to-finish workflow.

---

# The OpenStack Master Lab: Installation to Instance Launch

## Phase 1: Conceptual Architecture

To understand OpenStack, it helps to map its components to the AWS services you might already know:

| OpenStack Component | AWS Equivalent | Function |
| :--- | :--- | :--- |
| **Nova** | EC2 | Compute and virtual machine management |
| **Neutron** | VPC | Networking, subnets, and routing |
| **Glance** | AMI | VM image storage and retrieval |
| **Cinder** | EBS | Block storage volumes |
| **Keystone** | IAM | Identity, authentication, and authorization |
| **Horizon** | AWS Console | Web-based user interface dashboard |

---

## Phase 2: Install OpenStack (DevStack)

*Note: Run this on a freshly deployed Ubuntu 22.04 or 24.04 instance.*

**1. Prepare the System and User**
DevStack requires a dedicated user with passwordless sudo access.

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git python3-pip
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
sudo su - stack
```

**2. Clone DevStack**
Always use the stable branch.

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
git checkout master
```

**3. Configure and Run**
Identify your primary network interface name using `ip addr show`. Then, create the `local.conf` file inside the `/opt/stack/devstack` directory:

```bash
cat > local.conf <<'EOF'
[[local|localrc]]
ADMIN_PASSWORD=secret123
DATABASE_PASSWORD=secret123
RABBIT_PASSWORD=secret123
SERVICE_PASSWORD=secret123
FLOATING_RANGE=192.168.1.224/27
FIXED_RANGE=10.11.12.0/24
FIXED_NETWORK_SIZE=256
FLAT_INTERFACE=ens5    # <-- REPLACE WITH YOUR AWS ETH0/ENS5 INTERFACE
LOGFILE=/opt/stack/logs/stack.sh.log
VERBOSE=True
LOG_COLOR=True
IMAGE_URLS="http://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img"
EOF
```

Run the installer (this takes 20-45 minutes):
```bash
sudo mkdir -p /opt/stack/logs && sudo chown -R stack:stack /opt/stack/logs
./stack.sh
```

---

## Phase 3: Identity & Resource Preparation

Once Horizon is running, you must configure a project, user, and resources. Never work directly in the default `admin` project.

**1. Create Project and User**
```bash
source /opt/stack/openrc admin admin

openstack project create --description "Cloud Lab" --enable CloudLab-Project
openstack user create --project CloudLab-Project --password "Student@123" --email student01@lab.local --enable student01
openstack role add --project CloudLab-Project --user student01 member
```

**2. Upload an Image (Glance)**
We will use Ubuntu 22.04.

```bash
cd /tmp
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img
openstack image create --container-format bare --disk-format qcow2 --file /tmp/jammy-server-cloudimg-amd64.img --public --min-ram 512 --min-disk 8 "Ubuntu 22.04 LTS"
```

**3. Define a Flavor (Hardware Spec)**
```bash
openstack flavor create --vcpus 1 --ram 1024 --disk 10 --public lab.small
```

---

## Phase 4: Configure Networking (Neutron)

Neutron handles your OpenStack VPC equivalent. Switch to your user project context first:

```bash
export OS_PROJECT_NAME=CloudLab-Project
```

**1. Create Private Network**
```bash
openstack network create private-net
openstack subnet create --network private-net --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 --dns-nameserver 8.8.8.8 --ip-version 4 private-subnet
```

**2. Attach Router**
Connect your private network to the public (external) network provided by DevStack.

```bash
source /opt/stack/openrc admin admin
openstack router create lab-router
openstack router set --external-gateway public lab-router
openstack router add subnet lab-router private-subnet
```

**3. Prepare Security Group**
Allow SSH and Ping traffic.

```bash
source /opt/stack/openrc student01 Student@123
openstack security group create lab-sg
openstack security group rule create --protocol tcp --dst-port 22 --remote-ip 0.0.0.0/0 lab-sg
openstack security group rule create --protocol icmp --remote-ip 0.0.0.0/0 lab-sg
```

---

## Phase 5: Launch the Instance (Nova)

With the foundation built, you can now launch your VM.

**1. Generate Keypair**
```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
openstack keypair create lab-keypair > ~/.ssh/lab-keypair.pem
chmod 400 ~/.ssh/lab-keypair.pem
```

**2. Launch Server**
Retrieve your network ID and boot the VM.

```bash
NETWORK_ID=$(openstack network show private-net -f value -c id)

openstack server create \
  --flavor lab.small \
  --image "Ubuntu 22.04 LTS" \
  --network $NETWORK_ID \
  --security-group lab-sg \
  --key-name lab-keypair \
  --wait \
  my-first-instance
```

**3. Assign a Floating IP**
To access the VM from outside its internal network, it needs a Floating IP (similar to an AWS Elastic IP).

```bash
source /opt/stack/openrc admin admin
openstack floating ip create public
# Assuming the output gave you an IP, e.g., 192.168.1.105
openstack server add floating ip my-first-instance 192.168.1.105
```

You can now track your instance's status using `openstack server list` and view boot logs via `openstack console log show my-first-instance`.
