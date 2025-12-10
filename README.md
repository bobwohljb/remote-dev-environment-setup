# Remote Development Environment Setup Guide

This guide provides step-by-step instructions for setting up a remote development environment using Docker and connecting to it via SSH. This setup is compatible with JetBrains Toolbox for remote development.

## Prerequisites

- Docker installed on your local machine
- Basic knowledge of terminal/command line
- SSH client

## Step 1: Create SSH Key for Remote Development

1. Generate a new SSH key specifically for remote development:

```bash
ssh-keygen -t ed25519 -C "remote-dev" -f ~/.ssh/id_ed25519_remote
```

2. When prompted for a passphrase, you can either set one or leave it empty for passwordless login.

3. The key will be created at:
   - Private key: `~/.ssh/id_ed25519_remote`
   - Public key: `~/.ssh/id_ed25519_remote.pub`

4. View your public key (you'll need this later):

```bash
cat ~/.ssh/id_ed25519_remote.pub
```

## Step 2: Build the Docker Image

1. Create a directory for your remote development setup:

```bash
mkdir -p remote-dev-setup
cd remote-dev-setup
```


2. Create a new file named `Dockerfile` in your project directory with the following content:

```dockerfile
FROM ubuntu:22.04

# Avoid interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install essential development tools
RUN apt-get update && apt-get install -y \
    openssh-server \
    sudo \
    curl \
    git \
    vim \
    openjdk-17-jdk \
    python3 \
    python3-pip \
    nodejs \
    npm \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Create a development user
RUN useradd -m -s /bin/bash dev && \
    echo 'dev:password123' | chpasswd && \
    usermod -aG sudo dev

# Configure SSH
RUN mkdir /var/run/sshd && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config


# Set up workspace directory
RUN mkdir -p /home/dev/workspace && \
    chown dev:dev /home/dev/workspace

# Switch to dev user for final setup
USER dev
WORKDIR /home/dev

# Switch back to root for the CMD
USER root

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

Make sure the Dockerfile saved is in this directory.

3. Build the Docker image:

```bash
docker build -t my-remote-dev:latest .
```

> **Note:** The password for the 'dev' user is set to `password123` in the Dockerfile.

## Step 3: Run the Docker Container

1. Create and start the container:

```bash
docker run -d -p 2222:22 \
  -v $(pwd)/workspace:/home/dev/workspace \
  --name remote-dev my-remote-dev
```

This command:
- Runs the container in detached mode (`-d`)
- Maps port 2222 on your host to port 22 in the container
- Mounts the `workspace` directory from your current directory to `/home/dev/workspace` in the container
- Names the container `remote-dev`

## Step 4: Set Up SSH Access

### Option 1: Password Authentication

You can connect using the password set in the Dockerfile:

```bash
ssh dev@localhost -p 2222
```

When prompted, enter the password: `password123`

### Option 2: SSH Key Authentication (Recommended)

1. Connect to the container:

```bash
docker exec -it remote-dev bash
```

2. Switch to the dev user:

```bash
su - dev
```
or
```bash
sudo - dev
```

3. Create the SSH directory and set permissions:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

4. Add your public key to authorized_keys:

```bash
cat >> ~/.ssh/authorized_keys
```

5. Paste your public key (from Step 1.4), press Enter, then press Ctrl+D to save.

6. Set proper permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

7. Verify the setup:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

8. Exit the container:

```bash
exit
exit
```

## Step 5: Connect to the Container via SSH

Connect using your SSH key:

```bash
ssh -i ~/.ssh/id_ed25519_remote -p 2222 dev@localhost
```

## Step 6: Simplify SSH Connection (Optional)

Add this to your `~/.ssh/config` file:

```
Host remote-dev
    HostName localhost
    Port 2222
    User dev
    IdentityFile ~/.ssh/id_ed25519_remote
```

Now you can connect simply by typing:

```bash
ssh remote-dev
```

## Step 7: Connect Using JetBrains Toolbox

1. Open JetBrains Toolbox
2. Select "Connect to SSH"
3. Enter the SSH configuration:
   - Host: `remote-dev` (if you set up the SSH config) or `localhost`
   - Port: 2222 (if not using SSH config)
   - User: dev
   - Authentication: Key pair
   - Private key file: `~/.ssh/id_ed25519_remote`

4. Connect and select the IDE you want to use

## Troubleshooting

### Container Not Running

Check the status of your container:

```bash
docker ps -a
```

If the container exists but is not running, start it:

```bash
docker start remote-dev
```

### Remove and Recreate Container

If you need to remove the container:

```bash
docker rm -f remote-dev
```

Then recreate it using the command from Step 3.

### Connection Issues

If you're having trouble connecting, ensure:
1. The container is running
2. You're using the correct port (2222)
3. SSH key permissions are correct (600 for private key, 644 for public key)
4. The authorized_keys file in the container has the correct permissions (600)
