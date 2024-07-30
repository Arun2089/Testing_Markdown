# Ansible x AWS [VPC,Load Balancer] x Docker

# TASKs

  # 1. Create 3 Private Servers On AWS
    - Create new VPC in AWS
    
           Name = My_VPC IPV4 CIRD = 10.0.0.0/16 (Gives Approx 65K ips)
    
    - Create 1 Public Subnet and 3 Private Subnet in My_VPC
    
          My_Public_Subnet  -  ipv4 subnet cidr - 10.0.1.0/26 [256 ips] in Availability Zone 1a
    
          My_Private_Subnet1 -  ipv4 subnet cidr - 10.0.2.0/26 [256 ips] in Availability Zone 1a
    
          My_Private_Subnet2  -  ipv4 subnet cidr - 10.0.3.0/26 [256 ips] in Availability Zone 1b
    
          My_Private_Subnet3  -  ipv4 subnet cidr - 10.0.4.0/26 [256 ips] in Availability Zone 1c
    
          ps. [ Make Sure The Private Subnets Are In Different Availability Zones or AZs]
    
        
    
    - Create 1 Internet Gateway [IGW]
    
           Attach To My_VPC
    
    - Create 1 NAT Gateway and Associate it with the public subnet
    
           Name - My_NAT 
    
           Select My_Public_Subnet  as Subnet
    
           Allocate A Elastic Ip 
    
    - Create Route Tables for All Subnets,
    
           for My_Public_Subnet  route 0.0.0.0/0 to IGW [edit routes]
    
            **Explicit subnet associations -**My_Public_Subnet **[subnet association ]**
    
           for My_Private_Subnet123 route 0.0.0.0/0 to NAT [edit routes]
    
           **Explicit subnet associations -**My_Private_Subnet123 **[subnet association ] [ For all 3 ]**
    
           ps. total 4 routes tables will be created [for each subnet]
    
    - Create 3 instance without public-IP and select the Created VPC and Private Subnet 1,2,3
    
          3 instances will be created in different Subnets
    
    Private_ubuntu_1 -    My_VPC - My_Private_Subnet_1 
    
    Private_ubuntu_2 -    My_VPC - My_Private_Subnet_2
    
    Private_ubuntu_3 -    My_VPC - My_Private_Subnet_3 
    
    - Create 1 instance with public-IP and select the Created VPC and Public Subnet
        
        Public_ubuntu -    My_VPC - My_Public_Subnet 
        
        ![VPC Resource Map Should Look Like This](Ansible%20x%20AWS%20%5BVPC,Load%20Balancer%5D%20x%20Docker%20957e416d9ecd489fa62e80650278d29f/Screenshot_2024-07-30_at_12.06.34_AM.png)
        
        VPC Resource Map Should Look Like This
        
 # 2. Config SSH And Test Connection Between The Instances
    - In You Local Machine Go to ~/.ssh/config , and write the config for private and public servers
    
    ```bash
    Host jump-server
    	HostName <Public_IP_of_Public_Instance>
            Port 22
    	IdentityFile /path/to/key.pem/for/instance/login
    	User ubuntu
    
    Host private1
    	HostName <Private_IP_of_Private_Instance_1>
    	IdentityFile ~/Downloads/Key.pem
    	Port 22
    	User ubuntu
    	ProxyJump jump-server
    
    Host private2
    	HostName <Private_IP_of_Private_Instance_2>
    	IdentityFile /path/to/key.pem/for/instance/login
    	Port 22
    	User ubuntu
    	ProxyJump jump-server
    
    Host private3
    	HostName <Private_IP_of_Private_Instance_3>
    	IdentityFile /path/to/key.pem/for/instance/login
    	Port 22
    	User ubuntu
    	ProxyJump jump-server
    ```
    
    - Added The  3 private servers and 1 public server [the public servers works as Bastion Server, to access the private server]
    - Test SSH to all the instances
    
    ```bash
    ssh jump-server 
    ssh private1
    ssh private2
    ssh private3
    ```
    
 # 3. Install Ansible On Local Machine
    - install pipx
    
    ```bash
    brew install pipx
    ```
    
    - install ansible by pipx
    
    ```bash
    $ pipx install --include-deps ansible
    ```
    
 # 4. Create Hosts File For Ansible Host [Nodes - (instances - Private_ubuntu_1,2,3)]
    - Create  a hosts File and add the ssh Host [create this file in the same directory as the ansible_playbook.yml is present in  or the path you want to run ansible commands on]
    
    ```bash
    [private_servers]
    private1
    private2
    private3
    
    ```
    
 # 5. Create Load_Balancer [AWS Service] To Host [access] The Nginx Running on Private Host.
    - Create A Target Group Named [My_Target_Group]
    
          Select Target Type as  Instances 
    
          Select My_VPC 
    
          Select All 3 Private_ubuntu_1,2,3 Instances
    
    - Create A Load Balancer
    
          Application Load Balance [internet-facing]
    
            Select My_VPC
        
            Select Subnets Present In Different Availability Zones
    
             ps. Minimum 2 Different Availability Zones Should Be Selected [Thats Why We Created 3  Different Private_Subnet In Different Availability Zones]
    
              My_Private_Subnet_1 - ap-south-1a
        
              My_Private_Subnet_2 - ap-south-1b
    
              My_Private_Subnet_3 - ap-south-1c
    
            In Listener HTTP: 80  [because the nginx on private server is hosted on 80 by default ]
    
            Select target group [My_Target_Group]
    
    ![The Load Balancer Resource Map Should Look Like This](Ansible%20x%20AWS%20%5BVPC,Load%20Balancer%5D%20x%20Docker%20957e416d9ecd489fa62e80650278d29f/Screenshot_2024-07-30_at_1.32.58_AM.png)
    
    The Load Balancer Resource Map Should Look Like This
    
    Hit The DNS name In The Load Balancer Details [to Server The Hosted Content on Private Servers By Nginx] 
    
 # 6. Write a Ansible_Playbook
    - To Deploy Nginx on Private_servers
    - With Different Content And Info About The Host
    - Install Docker
    - Pull an Nginx Image
    - Run a Container From The Nginx Image and Expose 3000 Port To Host
    - Send Telegram Message If All Work Correctly [Bot_father and Raw_data_Bot used ]
    
    ```yaml
    ---
    - name: Install and configure Nginx and Docker
      hosts: private_servers
      vars:
          chat_id: "1187236965"
          token: "7476406886:AAEqSf6RzsylrKQxfl_00-bAycthlJUBKDc"
          load_balancer_dns_url: "http://load-balancer-for-private-1126193604.ap-south-1.elb.amazonaws.com/"
      become: yes
    
      tasks:
        - name: apt-get update
          apt:
            update_cache: yes
    
        - name: Install Nginx
          apt:
            name: nginx
            state: present
    
        - name: Start and enable Nginx
          service:
            name: nginx
            state: started
            enabled: yes
    
        - name: Create Different HTML for all
          copy:
            dest: /var/www/html/index.html
            content: |
              Arun Lohar <br> Host Name = {{ inventory_hostname }} <br> Private IP = {{ ansible_default_ipv4.address }} <br> Operating System ={{ ansible_distribution }} <br> User = {{ ansible_user_id }}
    
        - name: Install dependencies for Docker
          apt:
            name: 
              - apt-transport-https
              - ca-certificates
              - curl
              - software-properties-common
            state: present
    
        - name: Add Docker’s official GPG key
          apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present
    
        - name: Add Docker repository
          apt_repository:
            repo: deb [arch=amd64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable
            state: present
    
        - name: Update apt cache after adding Docker repository
          apt:
            update_cache: yes
    
        - name: Install Docker
          apt:
            name: docker-ce
            state: present
    
        - name: Start and enable Docker
          service:
            name: docker
            state: started
            enabled: yes
    
        - name: Pull Nginx Docker image
          docker_image:
            name: nginx
            source: pull
    
        - name: Run Nginx container with port 3000 exposed
          docker_container:
            name: nginx_container
            image: nginx
            ports:
              - "3000:80"
            state: started
            restart_policy: always
    
      
        - name: Sending Message To Telegram For Success 
          command: curl -I https://api.telegram.org/bot{{token}}/sendMessage?chat_id={{chat_id}}&text=PlayBook%20Executed%20Successfully%20on%20Server%20={{inventory_hostname}},%20Private%20IP={{ansible_default_ipv4.address}},%20Operating%20System%20={{ansible_distribution}},%20User%20=%20{{ansible_user_id}},Visit-%20{{load_balancer_dns_url}}
          register: curl_result
    
       
    ```
