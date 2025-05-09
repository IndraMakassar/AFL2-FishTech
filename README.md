# AFL 2 DEPI
#### 0806022210004 Antonius Indra Dharma Prasetya
#### 0806022210009 Anugrah Mariani Pirade`
#### 0806022210012 Enrico Kevin Ariantho

## Architecture Diagram
![image](https://github.com/user-attachments/assets/0a44d685-b05b-479b-87b7-23538064ff3b)


## AWS Setup Instructions
## 🖥️ Step 1: Setup & Launch AWS EC2 (Ubuntu)
1. **Create an EC2 Instance**
   - Choose Ubuntu AMI.
   - Create or select an existing key pair.

2. **Configure Security Groups**
   Create the following rules:

   | Type         | Port | Source           | Function            |
   |--------------|------|------------------|---------------------|
   | SSH          | 22   | IP Developer     | Remote access       |
   | HTTP         | 80   | 0.0.0.0/0        | Web service         |
   | HTTPS        | 443  | 0.0.0.0/0        | SSL                 |
   | PostgreSQL   | 5432 | EC2 IP           | Database connection |
   | Custom TCP   | 3000 | IP Developer     | Supabase Studio     |
   | Custom TCP   | 8000 | IP Developer     | REST API            |
   | Custom TCP   | 4000 | 0.0.0.0/0        | WebSocket API       |


3. **Adjust Storage**
   - Set root volume to 30 GB.

4. **Launch the Instance**

---

## 💻 Step 2: Configure Ansible on Control Node
1. **Find EC2 Public IP**
   - From AWS Console → EC2 → Instances → Details Active Instances.

2. **Generate SSH Key on WSL (if needed for passwordless SSH)**
   ```
   ssh-keygen
   ```
   ```
   ssh-copy-id <ansible_user>@<managed_node_ip> 
   ```
3. **Create or open the ansible inventory file**
   ```
   sudo nano /etc/ansible/hosts
   ```
   and fill with:
   ```
   [servers]
    node1 ansible_host=<managed_node_ip> ansible_user=<your_user> ansible_ssh_private_key_file=<path_to_ssh_key> ansible_port=22
   ```
4. **Check ansible connection**
   ```
   ansible all -m ping
   ```
   //picture ansible green

   
---


## 🖥️ Step 3: Make an Ansible Playbook & Execute the Ansible Playbook in Managed Node 
1. **Make a playbook for install docker and execute it**
   ```
   sudo nano install_docker.yml
   ```
   and fill with:
   ```
    - name: Install Docker on Managed Node
    hosts: servers
    become: yes
    tasks:
        - name: Install prerequisites
        apt:
            name: ["apt-transport-https", "ca-certificates", "curl", "software-properties-common"]
            state: present

        - name: Add Docker GPG Key
        apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present

        - name: Add Docker Repository
        apt_repository:
            repo: deb https://download.docker.com/linux/ubuntu focal stable
            state: present

        - name: Install Docker
        apt:
            name: docker-ce
            state: present

        - name: Start and Enable Docker
        service:
            name: docker
            state: started
            enabled: yes
   ```
   run:
   ```
   ansible-playbook install_docker.yml
   ```

2. **Make a playbook for install docker compose and execute it**
   ```
   sudo nano install_dockerCompose.yml
   ```
   and fill with:
   ```
    - name: Install Docker Compose on Ubuntu AWS server
    hosts: servers
    become: yes
    tasks:

        - name: Install required dependencies
        apt:
            name: curl
            state: present
            update_cache: yes

        - name: Download Docker Compose binary (Linux x86_64)
        get_url:
            url: https://github.com/docker/compose/releases/download/v2.24.5/docker-compose-linux-x86_64
            dest: /usr/local/bin/docker-compose
            mode: '0755'

        - name: Verify Docker Compose installation
        command: docker-compose --version
        register: compose_version

        - name: Show Docker Compose version
        debug:
            var: compose_version.stdout
   ```
   and run:
   ```
   ansible-playbook install_dockerCompose.yml
   ```

3. **Make a playbook for clone backend supabase from github to managed node and execute it**
   ```
   sudo nano clone_supabase.yml
   ```
   and fill with:
   ```
    - name: Setup self-hosted Supabase on remote Ubuntu server (Docker already installed)
    hosts: servers
    become: true

    vars:
        supabase_repo: https://github.com/supabase/supabase
        project_dir: /home/ubuntu/supabase-project
        supabase_dir: /home/ubuntu/supabase

    tasks:
        - name: Ensure Docker SDK for Python is installed (for Ansible Docker modules)
        command: pip3 install docker --break-system-packages

        - name: Clone Supabase repository (shallow)
        git:
            repo: "{{ supabase_repo }}"
            dest: "{{ supabase_dir }}"
            depth: 1
            force: yes

        - name: Create project directory
        file:
            path: "{{ project_dir }}"
            state: directory
            owner: ubuntu
            group: ubuntu
            mode: '0755'

        - name: Copy Supabase Docker files to project directory
        copy:
            src: "{{ supabase_dir }}/docker/"
            dest: "{{ project_dir }}/"
            remote_src: true
        notify: Pull Supabase Images

        - name: Copy .env.example to .env
        copy:
            src: "{{ supabase_dir }}/docker/.env.example"
            dest: "{{ project_dir }}/.env"
            remote_src: true

    handlers:
        - name: Pull Supabase Images
        community.docker.docker_compose_v2_pull:
            project_src: "{{ project_dir }}"
   ```
   and run:
   ```
   ansible-playbook clone_supabase.yml
   ```

4. **Make a playbook for start the supabase service and execute it**
   ```
   sudo nano start_supabase.yml
   ```
   and fill with:
   ```
    - name: Start Supabase services
    hosts: servers
    become: true

    vars:
        project_dir: /home/ubuntu/supabase-project

    tasks:
        - name: Start Supabase containers
        community.docker.docker_compose_v2:
            project_src: "{{ project_dir }}"
            state: present
   ```
   and run:
   ```
   ansible-playbook start_supabase.yml
   ```
   ![image](https://github.com/user-attachments/assets/3cc0282b-cfbb-47d8-965f-807834a64dfe)
   ![image](https://github.com/user-attachments/assets/a0e82585-365d-4ca3-8a07-d2d776ef5376)
---

## 💻 Step 4: Make RDS in AWS
1. **Open Aurora and RDS in AWS and Create Database**
2. **Choose PostreSQL in engine option**
3. **Add the same security group as the EC2 Supabase in "existing VPC security group"**
4. **Create the database**


## Explanation of Choices
We chose Supabase because supabase is an open source backend infrastructure. That means we could deploy it in our own server if we chose to and don't deal with the security aspect of the backend.
For the databases it is Postgresql because it is the default on supabase.
