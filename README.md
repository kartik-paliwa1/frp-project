# frp-project

Detailed Guide: Exposing Local Services with fatedier/frp

This document provides a comprehensive guide on how to set up and utilize fatedier/frp to expose local services running behind Network Address Translators (NATs) or firewalls to the public internet.
## 1. Introduction to frp

frp is an open-source, high-performance reverse proxy that facilitates secure and efficient network tunneling. Its primary function is to enable access to services hosted on private networks (e.g., home networks, internal development environments) from external internet locations, overcoming the limitations imposed by NATs and firewalls.

The architecture of frp is based on a client-server model:

   frps (frp server): This component operates on a publicly accessible server (typically a Virtual Private Server or VPS) with a dedicated public IP address. It functions as the central relay, listening for connections from frpc clients and routing incoming internet traffic to the appropriate local services.

  frpc (frp client): This component resides on the local machine where the service intended for exposure is hosted. It establishes a persistent, secure tunnel to the frps server, through which all communication for the local service is multiplexed and relayed.

## 2. Prerequisites

To successfully implement frp, the following resources and foundational knowledge are required:

Publicly Accessible Server (VPS):
       A Virtual Private Server instance from a cloud provider (e.g., AWS EC2, DigitalOcean, Linode, Google Cloud Compute Engine).
   A Linux-based operating system (e.g., Ubuntu, Debian, CentOS). This guide uses Ubuntu commands.
        Secure Shell (SSH) access to the server with administrative privileges (sudo).
   
    
Local Machine:
        A personal computer (Windows, macOS, or Linux) where the service to be exposed (e.g., a web server, SSH server, game server) is running.
   
    
Command-Line Interface Familiarity:
        Proficiency in basic command-line operations, including directory navigation, file downloading (wget), archive extraction (tar), text editing (nano or equivalent), and executing commands with sudo.
    
    
A Service to Expose:
        For the purpose of this guide, a simple Python HTTP server will be used as the local service.

## 3. Server Setup (frps)

This section outlines the procedure for configuring and deploying the frps component on your public server.
3.1. Server Preparation

Access your VPS via SSH and ensure the system's package list and installed software are up-to-date:

    ssh your_user@your_server_public_ip
    sudo apt update
    sudo apt upgrade -y

3.2. frp Binary Download (Server)

frp is distributed as a self-contained executable. Download the binary appropriate for your server's architecture (commonly linux_amd64).
    Identify Latest Release: Navigate to the official frp GitHub Releases page: https://github.com/fatedier/frp/releases.

  Copy Download Link: Locate and copy the download URL for the linux_amd64.tar.gz archive (e.g., frp_0.58.0_linux_amd64.tar.gz).

  Download on Server: Execute the wget command, replacing the URL with the copied link:

    wget https://github.com/fatedier/frp/releases/download/vX.Y.Z/frp_X.Y.Z_linux_amd64.tar.gz


  Extract Archive: Unpack the downloaded file, which will create a directory containing the frp binaries and configuration templates:

    tar -zxvf frp_X.Y.Z_linux_amd64.tar.gz

  Relocate and Navigate: For organizational purposes, move the extracted directory and change into it:

    mv frp_X.Y.Z_linux_amd64 frp_server
    cd frp_server

  The current directory should now contain frps, frps.toml, frpc, and frpc.toml.

3.3. frps.toml Configuration

The frps.toml file defines the server's operational parameters.

  Open Configuration File: Use a text editor to open frps.toml:

    nano frps.toml

  Add/Modify Content: Populate the file with the following configuration. It is imperative to replace placeholder values with secure, unique strings.

    # frps.toml - Server configuration file
    bind_port = 7000 # Port frps listens on for frpc client connections.
                     # This port must be allowed through the server's firewall.

    # Authentication token: Essential for securing client-server communication.
    # This token must exactly match the one configured in frpc.toml.
    token = "YOUR_SECURE_FRP_TOKEN_HERE" # ***CHANGE THIS VALUE***

    # Dashboard settings: Provides a web interface for monitoring frp status.
    # Recommended for ease of management and debugging.
    dashboard_port = 7500 # Port for the dashboard web interface.
                          # This port must also be allowed through the server's firewall.
    dashboard_user = "admin"
    dashboard_pwd = "YOUR_SECURE_DASHBOARD_PASSWORD" # ***CHANGE THIS VALUE***

    # HTTP Virtual Host (VHost) settings: Enables frp to act as a reverse proxy for web services.
    # Ensure standard HTTP (80) and HTTPS (443) ports are open on the server's firewall if used.
    vhost_http_port = 80
    vhost_https_port = 443

    # Logging configuration: Useful for diagnostics and monitoring.
    log_file = "/var/log/frps.log" # Path to the server log file.
    log_level = "info" # Logging verbosity: debug, info, warn, error.
    log_max_days = 3 # Number of days to retain log files.

   Save and Exit: Press Ctrl+X, then Y, then Enter.

3.4. Running frps as a systemd Service

For reliable, persistent operation, it is recommended to run frps as a systemd service.

  Relocate Binaries and Configuration: Move the frps executable and frps.toml to standard system paths:

    sudo mv frps /usr/bin/frps
    sudo mkdir -p /etc/frp
    sudo mv frps.toml /etc/frp/frps.toml

  Create systemd Service Unit File:

    sudo nano /etc/systemd/system/frps.service

   Paste Service Configuration: Insert the following content into the file:

    # /etc/systemd/system/frps.service - systemd unit file for frps
    [Unit]
    Description=Frp Server Service
    After=network.target syslog.target
    Wants=network.target

    [Service]
    Type=simple
    # Specifies the full path to the frps executable and its configuration.
    ExecStart=/usr/bin/frps -c /etc/frp/frps.toml
    Restart=on-failure # Automatically restart frps if it crashes.
    RestartSec=10      # Wait 10 seconds before attempting a restart.
    ExecReload=/bin/kill -HUP $MAINPID # Command to gracefully reload the service.

    [Install]
    WantedBy=multi-user.target # Ensures frps starts automatically on system boot.

   Save and Exit: Press Ctrl+X, then Y, then Enter.

  Reload systemd and Manage Service:

    sudo systemctl daemon-reload # Reload systemd manager configuration.
    sudo systemctl enable frps   # Enable frps to start on boot.
    sudo systemctl start frps    # Start the frps service immediately.
    sudo systemctl status frps   # Verify the service status (should show "active (running)").

3.5. Server Firewall Configuration

Proper firewall configuration is critical for security and accessibility. The ports used by frps must be explicitly opened. ufw (Uncomplicated Firewall) is commonly used on Ubuntu.

  Allow SSH Access:

    sudo ufw allow ssh

  Allow bind_port: Permit incoming connections for frpc clients:

    sudo ufw allow 7000/tcp

   Allow dashboard_port (if enabled): Grant access to the frp dashboard:

    sudo ufw allow 7500/tcp

   Allow HTTP/HTTPS Ports (if vhost_http_port/vhost_https_port are used):

    sudo ufw allow 80/tcp
    sudo ufw allow 443/tcp

  Enable UFW: If not already active, enable the firewall:

    sudo ufw enable # Confirm with 'y' if prompted.

  Verify UFW Status: Confirm that the necessary ports are listed as ALLOW:

    sudo ufw status

## 4. Local Client Setup (frpc)

This section details the setup of the frpc component on your local machine and its configuration to expose a service.
4.1. Local Service Preparation (Example: Python HTTP Server)

On your local machine, open a terminal. Create an index.html file and start a simple Python HTTP server:

    # Create a simple index.html file
    echo "<h1>Hello from my local server!</h1>" > index.html

    # Start a Python HTTP server on port 8000
    python3 -m http.server 8000

Ensure this terminal remains open as the Python server must be running for frpc to expose it. The server will be accessible locally at http://localhost:8000.
4.2. frp Binary Download (Local Machine)

Download the frp binary that corresponds to your local machine's operating system and architecture.

   Identify Latest Release: Access the frp GitHub Releases page: https://github.com/fatedier/frp/releases.

  Copy Download Link: Download the appropriate file (e.g., frp_X.Y.Z_darwin_amd64.tar.gz for macOS, frp_X.Y.Z_windows_amd64.zip for Windows, frp_X.Y.Z_linux_amd64.tar.gz for Linux).

   Download and Extract:

   Linux/macOS:

        wget https://github.com/fatedier/frp/releases/download/vX.Y.Z/frp_X.Y.Z_YOUR_OS_ARCH.tar.gz
        tar -zxvf frp_X.Y.Z_YOUR_OS_ARCH.tar.gz
        mv frp_X.Y.Z_YOUR_OS_ARCH frp_client
        cd frp_client

   Windows: Download the .zip file, extract its contents to a dedicated folder (e.g., C:\frp_client), and open a Command Prompt or PowerShell within that directory.

4.3. frpc.toml Configuration

This file configures the frpc client to connect to frps and define the local service to be exposed.

  Open Configuration File: Open frpc.toml using a text editor:

   Linux/macOS: 

        nano frpc.toml

  Add/Modify Content: Populate the file with the following configuration. Ensure the token matches the one set in frps.toml.

    # frpc.toml - Client configuration file
    [common]
    server_addr = "YOUR_SERVER_PUBLIC_IP" # Public IP address of your frps server.
    server_port = 7000                   # Must match the bind_port configured in frps.toml.
    token = "YOUR_SECURE_FRP_TOKEN_HERE" # ***MUST MATCH*** the token from frps.toml for authentication.

    # Proxy configuration for exposing a local HTTP server
    [[proxies]]
    name = "my-web-server" # A unique identifier for this specific proxy.
    type = "http"          # Specifies the type of proxy (e.g., http, https, tcp, udp).
    local_ip = "127.0.0.1" # The IP address of the local service (usually localhost).
    local_port = 8000      # The port on which your local service is listening (e.g., Python server on 8000).

    # Optional: Custom domain configuration for HTTP/HTTPS proxies.
    # If you own a domain (e.g., myapp.example.com), configure its DNS A or CNAME record
    # to point to your frps server's public IP. Uncomment and specify your domain.
    # custom_domains = ["myapp.example.com"]

    # Optional: Enable TLS for secure communication between frpc and frps (recommended).
    # transport.tls.enable = true

  Save and Exit.

4.4. Running frpc
    Open New Terminal/Command Prompt: Ensure your local service (e.g., Python HTTP server) is still active in a separate terminal window.
    Navigate to Client Directory: Change the current directory to where you extracted frp_client.
    Start frpc: Execute the frpc binary:

  Linux/macOS: 
  
    ./frpc -c ./frpc.toml
      
  Windows:
  
    frpc.exe -c frpc.toml

The terminal will display logs indicating a successful connection to the frps server and the activation of the configured proxy.
5. Verification and Testing

With both frps and frpc operational, proceed to verify public accessibility of your exposed service.

  Access the frp Dashboard (Optional):

  From any internet-connected device, open a web browser.

  Navigate to http://YOUR_SERVER_PUBLIC_IP:7500 (substitute with your server's actual public IP).

  Log in using the dashboard_user and dashboard_pwd credentials defined in frps.toml.

  Confirm that your "my-web-server" proxy is listed with an "online" status. This confirms successful client-server communication.

  Access Your Exposed Service:
        If custom_domains was configured in frpc.toml and DNS records are correctly set:
            Open a web browser and navigate to http://myapp.example.com (replace with your configured custom domain).
        If custom_domains was NOT configured (and vhost_http_port on frps is 80):
            Open a web browser and go to http://YOUR_SERVER_PUBLIC_IP.
            Alternatively, frp may automatically assign a subdomain if subdomain_host is configured on the server (e.g., http://my-web-server.YOUR_SERVER_PUBLIC_IP.nip.io).
            If vhost_http_port on frps is set to a non-standard port (e.g., 8080), access the service via http://YOUR_SERVER_PUBLIC_IP:8080.

Upon successful connection, the "Hello from my local server!" message (or content from your specific local service) should be displayed in the browser. This confirms that your local service is now publicly accessible via frp.
