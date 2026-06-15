# AWS Honeypot
## Project Summary
This project documents the end-to-end design, deployment, and hardening of a cloud-native, high-interaction SSH honeypot using Cowrie on AWS. The primary objective is to capture and analyze real-time adversarial behavior within a completely isolated architecture. By shifting administrative access to a secure custom port and leveraging strict egress filtering via an AWS Security Group, the host is protected against exploitation. Telemetry is securely funneled off the public internet using an AWS VPC Endpoint (PrivateLink) and a centralized CloudWatch agent, providing a safe, invisible pipeline for deep-dive threat intelligence gathering and digital forensics analysis.

### Project Roadmap & Architecture Index

| Phase | Phase Focus | Documented Subsections |
| --- | --- | --- |
| **Phase 1** | **Network Architecture & Infrastructure** | * Custom VPC Design<br>

<br>* Public Subnet & Internet Gateway Routing<br>

<br>* Security Group Blueprint (Initial Inbound/Outbound) |
| **Phase 2** | **EC2 Provisioning & Initial Deployment** | * Choosing the AMI (Ubuntu Server)<br>

<br>* Key Pair Generation & Management<br>

<br>* Network Configuration & Instance Deployment |
| **Phase 3** | **Operating System Configuration & Port Hardening** | * Initial SSH Access & Environment Reconnaissance<br>

<br>* Modifying the SSH Socket for Honeypot Preparation<br>

<br>* Updating Security Group Rules for the Custom Management Port<br>

<br>* *Troubleshooting SSH Connection*<br>

<br>* Verifying the New Management Connection<br>

<br>* Modifying Network Access Controls for Package Installation<br>

<br>* Installing Core Dependencies and Development Tools |
| **Phase 4** | **Cowrie Installation & Environment Setup** | * Creating the Dedicated Honeypot User<br>

<br>* Cloning the Repository & Establishing the Python Environment<br>

<br>* Initializing the Configuration Files<br>

<br>* *Troubleshooting Low-Port Binding & Authbind Configuration* |
| **Phase 5** | **Cowrie Deployment, Exposure, & Attack Validation** | * Executing the Application Build and Adjusting Configuration Layers<br>

<br>* Exposing the Honeypot to the Public Internet<br>

<br>* Validating Inbound Attacks & Monitoring Live Logs |
| **Phase 6** | **CloudWatch Integration & Private Network Bridging** | * Provisioning the IAM Logging Role<br>

<br>* Attaching the Identity Profile to the Virtual Machine<br>

<br>* Constructing the Private VPC Endpoint Bridge<br>

<br>* Collecting Network Identifiers for Final Configuration<br>

<br>* Configuring the Centralized Logging Agent<br>

<br>* Staging and Installing the Agent Package<br>

<br>* Activating and Validating the Cloud Logging Pipeline |
| **Phase 7** | **Post-Deployment Troubleshooting & Live Attack Analysis** | * *Troubleshooting Stale PID Files and Twistd Permissions*<br>

<br>* Validating CloudWatch Alert Ingestion<br>

<br>* Project Closeout: Infrastructure Decommissioning & Cost Management<br>

<br>* Conclusion & Reflections |

---

## Phase 1: Foundations of Cloud Networking
To build an effective detection pipeline, we must first establish an isolated networking environment within AWS. Building this infrastructure manually, rather than relying on automated wizards, ensures full visibility over how traffic flows between our resources and the public internet. It gives us a granular look at the mechanics of cloud networking right from the start.

### Creating the Virtual Private Cloud (VPC)
Our first step is establishing an isolated virtual network where all our honeypot components will reside. This acts as our blast radius boundary, ensuring our lab traffic stays completely separated from any production services.
1. In the AWS Management Console, use the search bar at the top to search for **VPC** and select the purple VPC icon to open the dashboard.
2. Click the orange **Create VPC** button.
3. Under **VPC settings**, select **VPC only**.

> **Note on Cost and Control:** The *VPC and more* option is a convenient shortcut that automates infrastructure creation, but it automatically provisions additional resources like NAT gateways that can quickly incur unexpected charges. Choosing *VPC only* allows us to build strictly within the AWS Free Tier while gaining hands-on experience with each individual component.

4. Configure the following settings:
* **Name tag:** Enter `Honeypot` to keep our assets organized and easy to track across the console.
* **IPv4 CIDR block:** Select **IPv4 CIDR manual input**.
* **IPv4 CIDR:** Enter `10.0.0.0/16`. This private IP range gives us a large pool of 65,536 addresses. While we only need a few for this lab, using a standard /16 block mirrors real-world enterprise design and gives us plenty of room to scale out the architecture later.
* **IPv6 CIDR block:** Select **No IPv6 CIDR block**. We are strictly focusing on IPv4 traffic tracking for this project to keep our log analysis clean.
* **Tenancy:** Leave this as **Default** so we are using shared hardware, which keeps us within the free tier. Choosing dedicated tenancy runs our instances on single-tenant hardware and costs a lot more.
* **VPC baseline encryption:** Select **No extra encryption control**. Default settings are perfectly fine here since we don't have compliance requirements for this lab.

5. Click **Create VPC**.


### Provisioning the Public Subnet
With the broad network boundary established, we need to carve out a specific subnet where our honeypot instance will live. Subnets allow us to group resources based on security and routing needs. Because our honeypot needs to interact directly with the internet to catch attackers, we're setting up a public subnet.
1. From the left-hand menu of the VPC Dashboard, select **Subnets**, then click the orange **Create subnet** button.
2. Under **VPC ID**, select the `Honeypot` VPC created in the previous step. You'll see the `10.0.0.0/16` block display automatically below it, confirming we're building within our isolated network.
3. Scroll down to **Subnet settings** and configure:

* **Subnet name:** Enter `Honeypot_Public_Subnet` to keep our naming conventions clear, making it obvious that this specific tier is intended for public-facing resources.
* **Availability Zone:** Choose the zone closest to your geographic location. Selecting an AZ close to `us-east-1` or `us-east-2` is often ideal for testing, as new cloud features deploy there first, and it reduces latency during our analysis.
* **IPv4 subnet CIDR block:** Enter `10.0.1.0/24`. This isolates a smaller pocket of 256 addresses specifically for this public tier. It sits neatly inside our larger `10.0.0.0/16` block, leaving the rest of the network available if we want to add private log analysis tiers later.

4. Click **Create subnet**.


### Internet Gateway (IGW) Deployment
A VPC is completely isolated from the outside world by default, which is great for security but bad for a honeypot that needs to be found by scanners and attackers. To allow our honeypot to receive inbound malicious traffic and let us manage the box externally, we must attach an Internet Gateway to the VPC. This acts as the bridge between our private cloud network and the public internet.

1. On the left-hand menu, select **Internet gateways**, then click **Create internet gateway**.
2. Set the **Name tag** to `Honeypot-IGW` to distinguish it from the VPC itself. Leave the default tag settings as they are.
3. Click **Create internet gateway**.
4. Once created, the status will show as *Detached*, meaning it exists in our account but isn't connected to our infrastructure yet. Click the **Actions** dropdown menu at the top right and select **Attach to VPC**.
5. Select your `Honeypot` VPC from the list, then click **Attach internet gateway**. This links the gateway to our network edge, though we still need to tell our specific subnet how to route traffic through it.


### Custom Route Table & Subnet Association
Now we need to tell our subnet how to route its traffic. By default, every VPC comes with a main route table that only allows internal traffic. To safely expose our public subnet while keeping the rest of the VPC secure by default, we will build a custom route table that explicitly directs internet-bound traffic out through our new gateway.

1. On the left-hand menu, select **Route tables**, then click **Create route table**.
2. Configure the following:
* **Name:** Enter `Honeypot-RT` to make its purpose clear during later troubleshooting or auditing.
* **VPC:** Select the `Honeypot` VPC so this table is mapped to our specific lab environment.

3. Click **Create route table**.
4. With `Honeypot-RT` selected in the list, look at the bottom tabs, choose the **Routes** tab, then click **Edit routes**.
5. Keep the default local route (`10.0.0.0/16 -> Local`) intact. This is a critical permanent route that allows any future resources inside our VPC network to communicate with each other.
6. Click **Add route** and enter the following path to the public internet:
* **Destination:** Enter `0.0.0.0/0`. This represents a default route, meaning any IPv4 traffic that doesn't match the internal local network will be sent here.
* **Target:** Select **Internet Gateway** from the dropdown, then select the `Honeypot-IGW` you created. This tells the VPC exactly where to send that external traffic.

7. Click **Save changes**.
8. Switch to the **Subnet associations** tab right next to the Routes tab and click **Edit subnet associations**.
9. Check the box next to `Honeypot_Public_Subnet` and click **Save associations**. This is the exact step that officially makes our subnet "public" by explicitly tying it to the internet-bound route table we just configured.


### Defensive Firewalls: Security Group Configuration
To ensure we can manage the environment safely while letting the system operate correctly, we must establish tight network controls. AWS Security Groups act as stateful, host-level firewalls that inspect traffic right at the network interface of our instances. Unlike a standard production setup where you block the world, our honeypot needs to be wide open to attackers on certain ports later on. However, we have to protect our management access first and severely restrict outbound traffic to prevent our lab from becoming a liability.

1. On the left-hand menu, select **Security groups**, then click **Create security group**.
2. Configure the basic details:
* **Security group name:** Enter `Honeypot-SG` to identify this firewall policy easily.
* **Description:** Enter `Honeypot administration and essential outbound rules` so its purpose is clear to anyone auditing the environment.
* **VPC:** Select the `Honeypot` VPC to apply these firewall rules to our isolated network.

3. Review the **Inbound rules** configuration.

> **Configuration Note:** To maintain management access to the underlying instance, ensure an inbound rule explicitly allows SSH traffic restricted strictly to your administrative workstation. We'll open the actual "honeypot" ports directly inside the OS firewall configuration later, rather than opening port 22 to the world here. This keeps our actual management plane hidden.

* **Type:** Select **SSH**.
* **Port Range:** This defaults to `22`.
* **Source:** Select **My IP** from the dropdown. The console will automatically detect your public IP and append a `/32` subnet mask, ensuring your backend management access is protected from external scanning and brute force attempts.

4. Under **Outbound rules**, remove any default "all traffic open to everything" rules. Leaving outbound traffic wide open is a major risk if a box gets compromised. Add the following mandatory restriction instead:
* **Type:** Select **Custom UDP**.
* **Port Range:** Enter `53`.
* **Destination:** Select `0.0.0.0/0`.

> **Security Nuance:** Restricting outbound traffic is a critical component of endpoint hardening, especially for a honeypot. By opening port 53 via UDP to anywhere, we allow the system to perform essential DNS resolution so it can update packages and connect to AWS logging endpoints. At the same time, blocking all other outbound ports prevents the honeypot from being utilized by an attacker as a launchpad to scan other networks, download malicious tools, or join a DDoS botnet against the internet.

5. Click **Create security group**.

---

## Phase 2: Virtual Machine Provisioning & Instance Hardening
With our network architecture securely in place, we can now deploy the actual computing resource that will host our honeypot. Building this out requires selecting an appropriate operating system and sizing the instance correctly to stay within the AWS Free Tier while ensuring we have enough performance to log attacker activity.

### Launching the EC2 Instance
Deploying our virtual machine is where the project really starts getting fun. AWS offers a massive catalog of Amazon Machine Images (AMIs) ranging from enterprise platforms to niche distributions, which can easily lead to accidental costs if you aren't careful. For our honeypot, we want a stable, well-documented environment, making an Ubuntu Server LTS image the ideal choice. By pairing this with a free-tier eligible instance type like a `t3.micro` or `t2.micro`, we get the perfect balance of dual vCPUs and 1 GiB of RAM, which is more than enough horsepower to log incoming attack traffic without spending a dime.

1. In the AWS Management Console search bar, type **EC2** and select the service to open the EC2 Dashboard.
2. Click the orange **Launch instance** button on the right side of the screen.
3. Under **Name and tags**, set the **Name** to `Honeypot-Cowrie`. This maintains our naming convention and clearly identifies this virtual machine as our primary host for the honeypot software.
4. Scroll down to the **Application and OS Images (Amazon Machine Image)** section. Here you'll see a wide variety of commercial and open-source operating systems.
* **OS Selection:** Select **Ubuntu** from the Quick Start icons. While almost any Linux distribution can support this project, Ubuntu is ideal because of its extensive documentation, massive package repositories, and predictable stability for open-source security tools.
* **AMI Dropdown:** Choose the standard **Ubuntu Server 24.04 LTS (HVM)** or whichever version is explicitly marked as *Free tier eligible*.

> **Console Tip:** If you want to verify your options, you can click **Browse more AMIs** on the right. This opens a deeper menu where you can filter strictly by *Free tier eligible* to make absolutely sure you do not accidentally select a premium, paid image.

5. Configure the architecture and sizing:
* **Architecture:** Leave this set to **64-bit (x86)** or **amd64**. This is standard for most software packages and ensures compatibility with the dependencies we will install later.
* **Instance type:** Select **t3.micro** from the dropdown menu. If your chosen region only supports **t2.micro** for the free tier, that works perfectly fine too. For a basic honeypot running a single service, we do not need significant CPU or memory performance. The micro instances give us a balance of 2 vCPUs and 1 GiB of RAM, which easily handles simultaneous automated scanning traffic without incurring any monthly costs.

Here is the expanded section for the key pair configuration.

---

### Key Pair Generation
Defining the network architecture for our instance ensures it actually uses the custom, isolated environment we built in Phase 1. By default, AWS will try to dump new instances into a generic default VPC with automated rules that can expose your infrastructure or run up a bill. By explicitly mapping our virtual machine to our custom Honeypot VPC and public subnet, we maintain total control over the traffic boundaries and ensure our management access remains strictly locked down to our own IP address.

1. Scroll down to the **Key pair (login)** section.

> **Security Nuance:** AWS uses public-key cryptography to secure your initial administrative access. Instead of a weak, easily brute-forced password, you'll use a private key file to authenticate over SSH. This is a crucial security barrier since this instance will eventually be exposed to the public internet.

2. Click the link to **Create new key pair**.
3. Configure the following settings in the pop-up window:
* **Key pair name:** Enter an easily recognizable name like `Honeypot-Key` so you can quickly identify it later if you deploy more infrastructure.
* **Key pair type:** Select **RSA**. This is a highly secure, widely compatible standard for cryptographic operations.
* **Private key file format:** Select **.pem**. This format is universally accepted by OpenSSH clients across Mac, Linux, and modern Windows terminals.

4. Click the orange **Create key pair** button.
5. **Important Storage Action:** Your browser will automatically download the private key file. You must save this file in a secure, accessible directory on your local machine. AWS does not store a copy of your private key, so if you lose this file, you'll permanently lose backend management access to your instance.


#### **Extra Note**
If you're running a Linux VM locally as your management workstation and downloaded the `.pem` file onto a Windows host, you need to securely move that file into your Linux environment to use it. If you have VirtualBox Guest Additions or VMware Tools configured, a quick drag-and-drop into your VM terminal or folder structure works perfectly.

If that isn't set up, you can open your Linux terminal and use `scp` (Secure Copy Protocol) to pull the file directly from your Windows host, provided your local network routing allows it. Another seamless approach for Windows users is leveraging a GUI-based file transfer client like WinSCP or Cyberduck, which lets you connect to your local VM over SFTP and drop the file right into your home directory.


### Network Configuration & Instance Deployment
Before launching our virtual machine, we must hook it up to the isolated network infrastructure we built in Phase 1. By default, AWS will try to dump new instances into a generic default VPC. By explicitly mapping our virtual machine to our custom `Honeypot` VPC and public subnet, we maintain total control over the traffic boundaries and ensure our management access remains strictly locked down.
1. Under **Network settings**, click the **Edit** button in the top right corner of the section.
2. Configure the following settings to align with our custom network:
* **VPC:** Select the `Honeypot` VPC from the dropdown menu. This ensures the instance is contained within our isolated blast radius.
* **Subnet:** Choose `Honeypot_Public_Subnet`. This places our instance in the public-facing tier of our network where we want to catch traffic.
* **Auto-assign public IP:** Select **Enable**. This is an essential setting because it automatically provisions a public-facing IPv4 address for our instance upon startup, ensuring external scanners and automated bots can find our honeypot.

3. Under **Security groups (Our Firewall)**, select **Select existing security group**.
4. From the list, check the box next to `Honeypot-SG` that we configured earlier. This instantly applies our host-level firewall rules, locking down SSH access to your specific IP address and strictly limiting outbound traffic to DNS resolution.
5. Scroll to the bottom of the page and review the **Summary** panel on the right. Verify your operating system, instance sizing, and network details are correct, then click the orange **Launch instance** button.

---

## Phase 3: Operating System Configuration & Port Hardening
With our cloud infrastructure running, we need to shift our focus to the underlying operating system. Before we can deploy our honeypot software, we have to establish a secure management session and modify the default SSH daemon settings. By moving our true administrative SSH access to a custom, high-numbered port, we free up port 22 so our honeypot can later intercept and log inbound scanning traffic without locking us out of our own system.

### Initial SSH Access & Environment Reconnaissance
To begin configuring our cloud environment, we need to log in to our local management workstation and establish a secure shell connection to the remote AWS instance using the cryptographic key pair we generated earlier.
1. Open your terminal on your local management machine and navigate to the directory where you saved your private key file.
2. Before AWS allows you to authenticate, you must restrict the permissions on your private key file. If the file is too widely readable, SSH will reject it as a security risk. Run the following command to grant read-only permissions strictly to the file owner:
```bash
chmod 400 <your-honeypot-pem>
```

3. With the permissions secured, initiate the remote connection by targeting the default `ubuntu` administrative user and your instance's current public IP address:
```bash
ssh -i <your-honeypot-pem> ubuntu@<honeypots-current-public-IP>
```

4. Upon your first connection, the terminal will prompt you to verify the authenticity of the host by displaying its ECDSA key fingerprint. Type `yes` and press enter to add the instance to your local `known_hosts` file and establish the session.
5. Once inside, you can run a quick environment check to confirm your privileges:
* Running `whoami` confirms you're logged in as the standard `ubuntu` administrative user.
* Running `pwd` shows your current working directory is `/home/ubuntu`.
* Running `ls` returns no files, confirming a clean, default installation state.

### Modifying the SSH Socket for Honeypot Preparation
Our ultimate goal is to catch attackers attempting to brute-force port 22, the standard port for SSH. To achieve this without losing administrative access, we need to reconfigure the actual SSH daemon to listen on a non-standard, custom high port. This leaves port 22 wide open for our Cowrie deployment to bind to later. Modern Ubuntu installations manage SSH connections via systemd sockets rather than traditional flat configuration files alone, meaning we modify the socket behavior directly.
1. Open the systemd override configuration for the SSH socket by running:
```bash
sudo systemctl edit ssh.socket
```

2. The text editor will display a file where the middle section contains comments starting with `###`. Anything you type between the top of the file and those comment blocks will act as a drop-in configuration override.
3. Scroll through the commented section to locate the default `ListenStream` entries, which usually look like `ListenStream=22` or `ListenStream=[::]:22`. To shift our real management access, add the following text at the top of the file, replacing `22` with your chosen custom high port (preferably a high numbered port of your choice that's not easy to guess):
```ini
[Socket]
ListenStream=
ListenStream=<your-custom-high-port>
```

> **Configuration Detail:** Entering an empty `ListenStream=` line first is a necessary systemd convention. It clears out the default port 22 listener entirely before the next line binds the service to your new custom port.

4. Save the file and exit the text editor.
5. To apply these configuration changes safely without rebooting the virtual machine, execute the following two commands:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

* **Why we run daemon-reload:** This tells systemd to scan the entire filesystem for new or modified configuration files. Without this, the background manager has no idea we just edited the drop-in override file.
* **Why we run restart ssh.socket:** This actively terminates the active listener on port 22 and immediately spins up the socket daemon on your new custom port, officially freeing up port 22 for our future honeypot configuration.

> **CRITICAL WARNING ON SESSION MANAGEMENT:** Before you close your current terminal window or test your new configurations, **keep your active SSH session open**. If you made a typo or misconfigured the socket file and close your only connection, you'll be permanently locked out of the instance. Keeping your current session alive gives you a safety net to fix any mistakes. If you close it and the configuration is broken, you'll have to terminate the EC2 instance and restart the entire configuration from scratch.


### Updating Security Group Rules for the Custom Management Port
Now that we have our internal operating system listening on our new custom port, we have to tell AWS to let that traffic through its external firewall. If we try to open a new connection right now, the AWS Security Group will drop the traffic because it only knows about the original port 22 rule. We need to explicitly permit our management IP to reach this new custom port.
1. Navigate back to the AWS Management Console, open the EC2 Dashboard, and select **Security Groups** from the left-hand menu.
2. Click on `Honeypot-SG` to view its details.
3. In the lower pane, select the **Inbound rules** tab, then click the gray **Edit inbound rules** button on the right.
4. Leave your original port 22 SSH rule exactly as it is for now. This keeps our current safety net active while we test the new path.
5. Click **Add rule** and configure the new management pipeline:
* **Type:** Select **Custom TCP**.
* **Port range:** Enter the exact custom high port number you configured inside the systemd socket file.
* **Source:** Select **My IP** from the dropdown menu. This automatically locks down your true administrative entryway to your specific local workstation, keeping it hidden from the rest of the internet.
* **Description:** Enter `Custom SSH Port` to maintain a clear audit trail of what this firewall rule does.

6. Click the orange **Save rules** button.

At this stage, your AWS firewall is primed to allow traffic to both ports from your local IP address, setting us up to safely test the new connection before we finalize our port adjustments.


#### **Troubleshooting SSH Connection**
If you attempt to open a new terminal and connect through your custom high port, you might find that the connection times out or gets refused. This often happens if the initial interactive `systemctl edit` command didn't properly write the drop-in override file, or if the socket didn't bind globally to all interface addresses. To manually force the override and ensure the SSH service binds correctly, you can create the configuration structure directly from the command line.

1. Manually create the systemd override directory for the SSH socket service if it doesn't already exist:
```bash
sudo mkdir -p /etc/systemd/system/ssh.socket.d/
```

2. Use the `tee` command to write the explicit socket binding rules directly into a new `override.conf` file. This eliminates any text editor formatting issues and forces systemd to clear port 22 and listen globally on your new custom management port:
```bash
sudo tee /etc/systemd/system/ssh.socket.d/override.conf << 'EOF'
[Socket]
ListenStream=
ListenStream=0.0.0.0:<your-custom-high-port>
ListenStream=[::]:<your-custom-high-port>
EOF
```

3. With the override file manually placed, run a full reload of the systemd manager and restart both the socket listener and the underlying SSH daemon to commit the changes:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo systemctl restart ssh
```

4. To verify that your troubleshooting steps worked and the operating system is actively listening for your management traffic on the new port, run a socket statistics query:
```bash
sudo ss -tlnp | grep ssh
```

If everything is configured correctly, your output should show the service actively listening on all IPv4 (`0.0.0.0`) and IPv6 (`[::]`) interfaces for your custom port:
```text
LISTEN 0      4096            0.0.0.0:<your-custom-high-port>        0.0.0.0:* users:(("sshd",pid=xxxx,fd=x))
LISTEN 0      4096               [::]:<your-custom-high-port>           [::]:* users:(("sshd",pid=xxxx,fd=x))
```

### Verifying the New Management Connection
Before we jump into updating the system and pulling down the honeypot files, we need to verify that our new entry point actually works. Testing the connection before making any further structural changes ensures that we can reliably reach our machine through the new custom port.
1. Open a **brand new** terminal window on your local management workstation, leaving your original session running in the background as a safety net.
2. Execute the SSH command, this time explicitly specifying your custom high-numbered port using the `-p` flag:

```bash
ssh -i <your-honeypot-pem> ubuntu@<honeypots-current-public-IP> -p <your-custom-high-port>

```

3. Because the operating system sees this as a new socket connection, you'll be prompted to confirm the host authenticity again. Type `yes` and press enter.
4. Once you see the prompt change to `ubuntu@ip-10-0-1-x`, you have successfully verified that your custom administrative channel is working. You can now safely close your original port 22 safety-net terminal session.


### Modifying Network Access Controls for Package Installation
With our management access verified on the new port, we need to prepare the instance to pull down Cowrie and its dependencies. However, if we try to run system updates right now, the commands will hang or fail because our AWS Security Group outbound rules are locked down to port 53 for DNS only. To install Git, Python, and the necessary security packages, we must temporarily open up web traffic outbound channels.
1. Navigate back to the AWS Management Console and open the VPC Dashboard.
2. Select **Security Groups** from the left-hand menu and click on your `Honeypot-SG`.
3. Select the **Outbound rules** tab at the bottom and click **Edit outbound rules**.
4. Leave your existing DNS (UDP port 53) rule intact, and click **Add rule** twice to add the following necessary repositories access paths:
* **Rule 1 (HTTP):**
  * **Type:** Select **HTTP**.
  * **Port range:** This will automatically populate with `80`.
  * **Destination:** Select `0.0.0.0/0`.
  * **Description:** Enter `Allow outbound updates via HTTP` so its purpose is clear.
* **Rule 2 (HTTPS):**
  * **Type:** Select **HTTPS**.
  * **Port range:** This will automatically populate with `443`.
  * **Destination:** Select `0.0.0.0/0`.
  * **Description:** Enter `Allow secure outbound updates and GitHub cloning` to document the rule.

5. Click the orange **Save rules** button.

With these outbound paths open, our virtual machine can now reach the public internet to download updates, pull down Python dependencies, and clone the Cowrie project from GitHub.


### Installing Core Dependencies and Development Tools
With our outbound network paths open, our operating system can now pull down the underlying packages required to build, run, and secure Cowrie. Rather than manually installing packages one by one, we can run a single comprehensive string to fetch our version control tools, Python virtual environment assets, and the specialized utilities needed to bind services to low-numbered privileged ports.
1. Execute the following command to update your local package indexes and install all necessary dependencies simultaneously:
```bash
sudo apt update && sudo apt install git libssl-dev libffi-dev build-essential libpython3-dev python3-minimal authbind python3-venv -y
```

> **Dependency Breakdown:** > * `git`: Essential for cloning the Cowrie repository directly from GitHub.
> * `build-essential` & `libssl-dev`: Provides the compiler tools and development libraries needed if any Python dependencies require local compilation.
> * `python3-venv` & `python3-minimal`: Allows us to create an isolated Python virtual environment, keeping Cowrie's dependencies separated from the core OS files.
> * `authbind`: A critical security utility that allows our non-root Cowrie user to bind to privileged ports (like port 22) without requiring dangerous full root permissions.

2. Once the installation process completes, run a quick confirmation query against the Debian package manager database to verify that the core tools are present and correctly installed:
```bash
dpkg -l | grep -E "git|authbind|python3-venv"
```

If the installation was successful, your terminal will return a list showing the package name, the installed version, and a status indicator of `ii` (which stands for "installed ok, installed") on the far left of each row.

---

## Phase 4: Cowrie Installation & Environment Setup
With the underlying operating system prepared and hardened, we are ready to deploy the core honeypot application. Cowrie strictly refuses to run under a root or high-privilege administrative account to ensure that an attacker cannot exploit the honeypot software to gain control of the host operating system. To satisfy this security boundary, we must create a dedicated, low-privilege system user before downloading and configuring the application code.

### Creating the Dedicated Honeypot User
1. Create a secure, non-root user account named `cowrie`. We will disable the password field for this user so it cannot be logged into directly from the outside world:
```bash
sudo adduser --disabled-password cowrie
```

Press enter to accept the defaults for the user profile details when prompted, and confirm the configuration.
2. Switch from your administrative `ubuntu` session into the context of your new `cowrie` user:
```bash
sudo su - cowrie
```

You'll notice your terminal prompt changes to `cowrie@<ip>:~$`, and running `pwd` will confirm you're safely isolated inside the `/home/cowrie` directory.

### Cloning the Repository & Establishing the Python Environment
Now that we are operating under our restricted user account, we can pull down the source code and build out an isolated execution environment for the application dependencies.
1. Clone the official Cowrie project repository directly from GitHub into your home directory:
```bash
git clone http://github.com/cowrie/cowrie.git
```

2. Move into the newly created project folder:
```bash
cd cowrie
```

3. Initialize an isolated Python virtual environment. This prevents Cowrie's package requirements from interfering with or breaking the system-wide Python libraries:
```bash
python3 -m venv cowrie-env
```

4. Activate the virtual environment to shift your execution context:
```bash
source cowrie-env/bin/activate
```

Once executed, your terminal prompt will change to include the environment prefix: `(cowrie-env) cowrie@ip-10-0-1-x:~/cowrie$`.
5. Upgrade `pip` within the virtual environment to ensure compatibility with modern deployment packages:
```bash
pip install --upgrade pip
```

6. Install the definitive list of project dependencies required to run the emulation engine:
```bash
pip install -r requirements.txt
```

### Initializing the Configuration Files
Cowrie manages its operating parameters using layered configuration files. It ships with a default template containing the baseline settings, which we must copy and customize.
1. Create a working copy of the distribution configuration file. We copy this to a new file so our personal modifications are kept separate from the underlying defaults:
```bash
cp src/cowrie/data/etc/cowrie.cfg.dist src/cowrie/data/etc/cowrie.cfg
```

2. Open your new configuration file in a text editor to begin tailoring the honeypot's persona:
```bash
nano src/cowrie/data/etc/cowrie.cfg
```


#### **Troubleshooting Low-Port Binding & Authbind Configuration**
When operating inside the isolated virtual environment as the `cowrie` user, you'll find that attempting to execute configuration adjustments or preparing the engine to listen on port 22 fails. Linux natively protects well-known ports (0–1023) by requiring full root privileges to bind to them. Because our `cowrie` account is intentionally stripped of administrative permissions, we must temporarily step back out to our `ubuntu` admin context to provision an execution rule within `authbind`.
1. Drop out of your current low-privilege `cowrie` session to return to the administrative `ubuntu` prompt:
```bash
exit
```

You'll see your terminal prompt shift back to `ubuntu@<ip>:~$`.

2. Use administrative privileges to create a dedicated tracking file for port 22 inside the `authbind` configuration directory:
```bash
sudo touch /etc/authbind/byport/22
```

3. Change the ownership of that file directly to your honeypot user so that Cowrie can interact with the binding policy:
```bash
sudo chown cowrie:cowrie /etc/authbind/byport/22
```

4. Adjust the file permissions to allow execution access, ensuring the system reads this file as an authorized binding exception for our restricted user:
```bash
sudo chmod 755 /etc/authbind/byport/22
```

5. With the system exception firmly in place, switch back into your dedicated honeypot user account to resume setup:
```bash
sudo su - cowrie
```

6. Navigate back into your project folder and re-activate your Python virtual environment to restore your previous deployment context:
```bash
cd cowrie
source cowrie-env/bin/activate
```

Your prompt will return to `(cowrie-env) cowrie@<ip>:~/cowrie$`, and the application now possesses the structural rights it needs to claim port 22 without running as root.

---

## Phase 5: Cowrie Deployment, Exposure, & Attack Validation
With our environment dependencies satisfied and `authbind` configured, we are ready to officially launch the honeypot engine. In this phase, we will perform the initial build execution, modify Cowrie's internal routing properties to claim the now-vacant port 22, expose the instance to the wider internet, and verify that our detection mechanisms are actively recording malicious reconnaissance.

### Executing the Application Build and Adjusting Configuration Layers
1. Before starting the service for the first time, run an editable installation of the project package to ensure all module paths are correctly mapped within the virtual environment:
```bash
pip install -e .
```

2. Attempt a baseline initialization of the service using `authbind` to allow the unprivileged user to pass traffic rules:
```bash
authbind cowrie start
```

3. By default, Cowrie is configured to listen internally on port 2222. To verify where the application is currently binding, run a socket statistics query:
```bash
ss -tulpn | grep :22
```

Your output will show the underlying event-driven engine (`twistd`) listening strictly on port 2222:
```text
LISTEN 0      50               0.0.0.0:2222        0.0.0.0:* users:(("twistd",pid=xxxx,fd=x))
```

4. Since our goal is to intercept automated scanner traffic aimed at the standard SSH port, we need to stop the honeypot and activate our custom configuration file so the engine switches ports. Stop the service by running:
```bash
cowrie stop
```

5. Copy your customized configuration file out into the active runtime directory so the engine can parse your structural overrides:
```bash
cp src/cowrie/data/etc/cowrie.cfg etc/cowrie.cfg
```

> **Configuration Note:** Leaving the file inside `src/` keeps it as a template fallback. Moving it into the root `etc/` folder tells Cowrie to actively use these settings (including any adjustments you make to specify listening on port 22) during execution.

6. Relaunch the honeypot engine with the new configuration active:
```bash
authbind cowrie start
```

7. Re-verify the socket binding properties to ensure the port swap took effect successfully:
```bash
ss -tulpn | grep :22
```

The output should now confirm that `twistd` has successfully claimed port 22:
```text
LISTEN 0      50               0.0.0.0:22          0.0.0.0:* users:(("twistd",pid=xxxx,fd=x))
```


### Exposing the Honeypot to the Public Internet
Now that our system is internally primed to catch attackers on port 22, we must prepare the AWS firewall. To do this safely, we'll first strip away the temporary outbound lifelines we granted the system during package setup, and only *then* open up the inbound rules so malicious bots can hit our instance. This ensures the honeypot is completely contained the exact moment it goes live.
1. Navigate back to the AWS Management Console, open the EC2 Dashboard, and select **Security Groups** from the left-hand menu.
2. Select `Honeypot-SG`, click the **Outbound rules** tab at the bottom, and click **Edit outbound rules**.
3. Delete both the **HTTP (Port 80)** and **HTTPS (Port 443)** rules that we temporarily added earlier for downloading dependencies.

> **Security & Future Planning Nuance:** Hardening a honeypot means enforcing strict egress filtering. We want to completely block our unprivileged environment from initiating outbound web requests to prevent compromised code from reaching C2 servers or pulling down malicious tools. We'll come back to this section in a later phase to surgically re-enable a specific, locked-down HTTPS path for log streaming, but for now, ensure the only active outbound rule is your original **Custom UDP Port 53** rule pointing to `0.0.0.0/0`.

4. Click **Save rules**.
5. With the outbound boundary locked down, click the **Inbound rules** tab and click **Edit inbound rules**.
6. Locate your original port 22 entry (which was previously locked down strictly to your administrative IP address).
7. Change the **Source** dropdown menu from *My IP* to **Anywhere-IPv4** (`0.0.0.0/0`).

> **Critical Architecture Reminder:** This action officially opens up our fake SSH doorway to the entire internet. Because we previously moved our *real* administrative management access to a secure high port locked down to our IP, our true management plane remains completely safe from external scanning.

8. Click **Save rules**.


### Validating Inbound Attacks & Monitoring Live Logs
Once the firewall rule is committed, automated internet scanners will typically discover and probe the instance almost instantly.
1. To watch attack connections drop into the system in real time, monitor the tail end of the Cowrie application log file:
```bash
tail -n 20 var/log/cowrie/cowrie.log
```

Within minutes, you should see inbound connection handshakes and brute force attempts from known malicious scanning networks. For example, during initial deployment validation, an inbound probe hit the system from the IP address `192.95.10.204`. Querying this address against VirusTotal immediately returns a high severity score (such as 12/91) flags for active phishing, malware distribution, and automated scanning activities.

2. To perform a definitive black-box test of the honeypot's emulation capabilities from your local management machine (or a testing host like Kali Linux), attempt to log in as an attacker targeting the default root user over port 22:
```bash
ssh root@<ip> -p 22
```

3. **Troubleshooting Host Key Contamination:** If you previously connected to this same AWS public IP address during Phase 3 when it hosted your real SSH server, your local machine will throw a high-priority warning error stating that the remote host identification has changed. This is expected because the honeypot is now presenting a fake, emulated SSH host key. To clear your local cache and allow the connection to proceed, run:
```bash
ssh-keygen -f '~/.ssh/known_hosts' -R <ip>
```

Once the old signature is cleared, rerun your attack simulation command.

4. Once inside the honeypot shell, play the role of the adversary. Execute a few standard discovery commands to see how the system responds:
* Running `whoami` will return `root`, tricking the attacker into believing they have achieved full administrative compromise.
* Running other basic file system reconnaissance tools will return convincing, fake environments designed to keep the attacker engaged.

5. To review your simulated attack steps and trace the adversary's footprints, drop back into your main administrative user shell and parse the logged session data:
```bash
tail -n 20 /home/cowrie/cowrie/var/log/cowrie/cowrie.log
```

The output file will display exact JSON-formatted entries mapping your test login timestamp, the emulated commands you executed, and the fake username and password combinations attempted.

---

## Phase 6: CloudWatch Integration & Private Network Bridging
While local file logs are excellent for immediate verification, a resilient security architecture requires centralizing log data to prevent tampering if a host is compromised. In this phase, we will provision an Identity and Access Management (IAM) role to grant our virtual machine secure logging privileges, update our host firewall, and establish an isolated AWS VPC Endpoint. This allows our honeypot to securely stream log data directly to CloudWatch without exposing our backend traffic to the public internet.

### Provisioning the IAM Logging Role
AWS uses IAM roles to grant permissions to resources like EC2 instances without needing to embed hardcoded, risky access keys inside the virtual machine.
1. In the AWS Management Console search bar, type **IAM** and select the service to open the IAM Dashboard.
2. On the left-hand navigation pane under **Access management**, click **Roles**.
3. Click the orange **Create role** button on the right side of the screen.
4. Configure the **Trusted entity type**:
* Select **AWS service**.
* Under the **Service or use case** dropdown, select **EC2**.
* In the radio buttons that appear below, select **EC2** again, then click **Next**.

5. On the **Add permissions** screen, type `CloudWatchAgentServerPolicy` into the permissions policies search bar. Check the box next to the matching policy name from the filtered list, then click **Next**.
6. Finalize the role details on the review page:
* **Role name:** Enter `EC2-CloudWatch-Logging-Role`.
* **Description:** Enter `Allows EC2 instances to call AWS services on your behalf.`

7. Review the **Trust policy** section at the bottom of the page. The console automatically builds this block based on your initial selections. Verify that your JSON matches the standard trust configuration structure:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

8. Click **Create role**.

### Attaching the Identity Profile to the Virtual Machine
Now that the role exists within our AWS account, we must explicitly pin it to our active honeypot instance so the operating system can inherit the necessary API permissions.
1. Navigate back to the **EC2 Dashboard** and click **Instances (running)**.
2. Select your `Honeypot-Cowrie` instance by checking its box, or right-click the instance row directly.
3. Select **Security** from the context menu, then click **Modify IAM role**.
4. In the **IAM role** dropdown menu, select the `EC2-CloudWatch-Logging-Role` you created in the previous step.
5. Click **Update IAM role** to commit the permission profile to the running virtual machine.

### Constructing the Private VPC Endpoint Bridge
To securely transmit logs to CloudWatch without opening our instance's outbound rules to the public internet via ports 80 or 443, we will deploy an interface VPC Endpoint. This spins up a private, internal network bridge that securely routes traffic directly from our isolated VPC to the CloudWatch logging service over the AWS internal backbone.
1. Navigate to the **EC2 Dashboard**, select **Security Groups** on the left menu, and select your `Honeypot-SG`.
2. Click the **Outbound rules** tab and click **Edit outbound rules**.
3. Click **Add rule** to allow secure local logging traffic:
* **Type:** Select **HTTPS**.
* **Port range:** This will default to `443`.
* **Destination:** Enter your internal VPC CIDR block, which is `10.0.0.0/16`.
* **Description:** Enter `Allow private outbound logging traffic to the VPC endpoint`.

4. Click **Save rules**.
5. Navigate to the **VPC Dashboard** and select **Endpoints** from the left-hand menu.
6. Click the orange **Create endpoint** button.
7. Configure the endpoint settings:
* **Name tag:** Enter `Honeypot-CloudWatch-Bridge`.
* **Service category:** Select **AWS services**.
* **Services search:** Type `logs` into the search box and press enter. Select the interface service string corresponding to your active deployment region (for example: `com.amazonaws.us-east-2.logs`).
* **VPC:** Select your custom `Honeypot` VPC from the dropdown menu.
* **Additional settings:** **Uncheck** the box next to *Enable Private DNS name*. Set the **IP address type** to **IPv4**.
* **Subnets:** Select your active availability zone (such as `us-east-2a`) and choose your public honeypot subnet (`Honeypot_Public_Subnet`) from the dropdown.
* **IP address type:** Select **IPv4**.

8. Scroll to the bottom and click **Create endpoint**.

### Collecting Network Identifiers for Final Configuration
To wrap up our network bridging phase, we need to collect a few specific identifiers from our infrastructure that the logging agent will require later to route data properly.
1. On the VPC Dashboard, click **Your VPCs** on the left-hand menu.
2. Select your `Honeypot` VPC. In the details panel at the bottom, highlight and copy your unique **VPC ID** string (it will start with `vpc-`), and make a mental note of your active **DHCP options set** name.
3. Return to the **Endpoints** screen on the VPC menu and click on your newly active `Honeypot-CloudWatch-Bridge` endpoint.
4. In the details tab at the bottom, locate the **DNS names** list. Copy the very first, top long-form Endpoint DNS entry in the list (this string will contain your specific endpoint ID and region details). Paste this somewhere safe alongside your VPC ID; we will feed these directly into our logging agent configurations in the next steps.


### Configuring the Centralized Logging Agent
With our network paths verified and identifiers collected, we need to return to our administrative operating system context to create the configuration mapping file for the CloudWatch collection agent.
1. Ensure you're in your primary management session (operating under the administrative `ubuntu` user via your custom high-numbered SSH port).
2. Create the target deployment directory for the cloud logging configuration files:
```bash
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/bin
```

3. Open a new configuration file in your text editor:
```bash
sudo nano /opt/aws/amazon-cloudwatch-agent/bin/config.json
```

4. Populate the file with the following structured JSON block. Be sure to replace `PASTE_YOUR_ENDPOINT_HERE` with the exact long-form VPC Endpoint DNS name you copied at the end of the previous section:

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root",
    "endpoint_override": "https://PASTE_YOUR_ENDPOINT_HERE"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/home/cowrie/cowrie/var/log/cowrie/cowrie.json",
            "log_group_name": "Honeypot-Cowrie-Logs",
            "log_stream_name": "{hostname}-attacks",
            "retention-in-days": 7
          }
        ]
      }
    }
  }
}

```

5. Save the file and exit the text editor.


### Staging and Installing the Agent Package
Because we previously locked down our outbound network access, our virtual machine cannot download the agent installation file directly from AWS repository pools. Instead, we will download the `.deb` package to our local management workstation first, and then securely upload it directly to our instance using our custom port pipeline.
1. On your local management machine terminal, run the following secure copy command to upload the agent package into the instance's temporary directory, ensuring you specify your custom administrative high port using the uppercase `-P` flag:
```bash
scp -P <your-custom-high-port> -i <your-honeypot-pem> ./amazon-cloudwatch-agent.deb ubuntu@<ip>:/tmp/
```

Once the file transfer completes successfully, the terminal will print a validation line confirming `amazon-cloudwatch-agent.deb` was received.

2. Switch back to your active remote administrative session on the virtual machine and invoke the Debian package installer to deploy the application asset:
```bash
sudo dpkg -i -E /tmp/amazon-cloudwatch-agent.deb
```

The installer will create the necessary background service groups, initialize system accounts, and deploy the core binaries.

### Activating and Validating the Cloud Logging Pipeline
With the package deployed and our configuration profile manually placed, we can initialize the engine daemon.
1. Instruct the CloudWatch control utility to fetch your custom configuration profile file, ingest the routing parameters, and establish the background logging service:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
```

The utility will process your file structure, map the target file path `/home/cowrie/cowrie/var/log/cowrie/cowrie.json`, and automatically configure the necessary systemd symlinks to handle service permanence.

2. Run a status check against the agent environment to verify that the logging daemon is active and running successfully:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

If everything is configured correctly, the status block output will confirm that the engine is **running** and operating under the guidance of your manual configuration settings:
```json
{
  "status": "running",
  "starttime": "2026-06-15T22:15:00Z",
  "configstatus": "configured"
}
```
---

### Phase 7: Post-Deployment Troubleshooting & Live Attack Analysis
System integration frequently introduces environmental friction. During the firewall and logging agent deployment, underlying binary permissions can shift or stale process states can prevent the emulation engine from binding to its designated socket. This final phase covers the remediation steps required to stabilize the environment, verify real-time alert ingestion, and properly manage cloud infrastructure lifecycle costs.

#### **Troubleshooting Stale PID Files and Twistd Permissions**
If the honeypot engine suddenly drops offline or refuses to bind to port 22 after infrastructure modifications, the issue is typically caused by a stale Process ID (`.pid`) lock file or a permission mismatch on the virtual environment binaries.
1. From your administrative `ubuntu` user session, force-reset the `authbind` execution policies and pass ownership rights directly to the python execution binary:
```bash
sudo touch /etc/authbind/byport/22
sudo chmod 777 /etc/authbind/byport/22
sudo chown cowrie:cowrie /home/cowrie/cowrie-env/bin/twistd
sudo chmod +x /home/cowrie/cowrie-env/bin/twistd
```

> **Security Engineering Note:** Shifting the file permissions to `777` on the `authbind` port configuration is a diagnostic step to ensure that the unprivileged account has absolute clearance to manipulate the low-port socket.

2. Switch back into your dedicated honeypot account and navigate to the project root directory:
```bash
sudo su - cowrie
cd /home/cowrie/cowrie
```

3. Re-activate your virtual environment and query your package state to verify all underlying framework libraries are active and properly versioned:
```bash
source ../cowrie-env/bin/activate
pip list
```

4. If you attempt to launch the engine and encounter a startup failure, a lingering lock file from a previous ungraceful shutdown is likely blocking the daemon. Manually purge the stale process file:
```bash
rm -f twistd.pid
```

5. Launch the emulation engine directly using the newly provisioned execution binary path wrapped in `authbind`:
```bash
authbind twistd cowrie
```

6. Run a socket query to confirm the honeypot has successfully reclaimed the primary defensive boundary:
```bash
ss -tlnp
```

The output will confirm that the process is back in a healthy `LISTEN` state on port 22.

#### **Validating CloudWatch Alert Ingestion**
With the engine stabilized, you can perform a live verification of the complete log streaming pipeline.
1. Open a new terminal session on your local management workstation and deliberately trigger the honeypot entryway:
```bash
ssh root@<ip> -p 22
```

2. Navigate to the **AWS CloudWatch Dashboard**, select **Log groups** from the left-hand menu, and click on your `Honeypot-Cowrie-Logs` group.
3. Select the active log stream (e.g., `{hostname}-attacks`).

The CloudWatch console will render your live attack events in real time. If a peer or an external adversary conducts a penetration test against the instance, every interactive command, directory traversal, attempted file drop, and automated botnet password-spray attempt will be recorded instantly as structured JSON events safely indexed in the cloud ledger.

---

### Conclusion & Reflections
Building and deploying this high-interaction SSH honeypot bridges the gap between theoretical cloud architecture and defensive security operations. By taking a default AWS instance and transforming it into a hardened, isolated detection environment, this project demonstrates a deep dive into several core blue team and engineering domains:

* **Secure Network Architecture:** Enforcing strict egress filtering and implementing VPC Endpoints to ensure that even if an application layer is compromised, the underlying blast radius is completely contained.
* **Identity & Access Management:** Utilizing IAM roles to securely grant resource permissions without relying on risky, static access keys.
* **System Hardening & Port Manipulation:** Reconfiguring low-level systemd socket behaviors to safely separate the true administrative management plane from public-facing deception traps.
* **Centralized Log Auditing:** Constructing a private pipeline to stream live, structured JSON event data directly into cloud monitoring pools for real-time analysis.

#### **Final Thoughts**
Watching an adversary interact with a system you built is one of the most effective ways to understand threat actor behavior. Whether analyzing automated botnet password sprays or tracing the deliberate, hands-on commands of a peer conducting a penetration test, the telemetry captured here provides raw insight into the modern threat landscape.

Ultimately, defensive security isn't just about building walls; it's about visibility, containment, and understanding the adversary's playbook. This project serves as a functional blueprint for deploying cloud-native deception technology safely, efficiently, and with total control over the data pipeline.


