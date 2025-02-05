# 06 Setting Up Jaynes Manager [Experimental]

From within a firewall protected server, launch the manager server `jaynes.server` then use ssh reverse tunnel to expose the local point via a public IP. The easiest way to do this is to ngrok. Look up how to install and run ngrok -- use it to listen to the running manager process.

## Method 1: Easy Setup with `ngrok`

The easiest way to setup is to download ngrok and use that to build a reverse tunnel.

1. First start the jaynes.server on the firewall protected server
   
    ```bash
    python -m jaynes.server --port <your-jaynes.server-port> --host 0.0.0.0
    ```
    
    The `--host 0.0.0.0` option allows the sanic server to accept request from all ip addresses.
    
    If you want this server to run in the background, you can use screen's daemon mode:
    
    ```bash
    screen -dm python -m jaynes.server --port <jaynes.server-port> --host 0.0.0.0
    ```
    
    To learn how to use screen, try to google for common screen commands.
    
3. Now run `ngrok`:
    
    ```bash
    ngrok tcp <your-jaynes.server-port>
    ```
    

## Method 2: Reverse Tunneling to Your Own EC2 Server

1. On aws server: modify the ssh deamon's config file, to expose the local tunnel directly as public gateways
    
    
    ```bash
    $ sudo vim /etc/ssh/sshd_config
    # /etc/ssh/sshd_config
    
    #...other config options 
    GatewayPorts yes
    ```

2. Then  start the jaynes server on the firewall protected server

    ```bash
    conda activate torch
    python -m jaynes.server --port <jaynes.server-port> --host 0.0.0.0
    ```

    The `--host 0.0.0.0` option allows the sanic server to accept request from all ip addresses.
    
    If you want this server to run in the background, you can use screen's daemon mode:
    
    ```bash
    screen -dm python -m jaynes.server --port <jaynes.server-port> --host 0.0.0.0
    ```
    
    To learn how to use screen, try to google for common screen commands.

3. Now you need to build the tunnel. If you missed step 1, your the remote port `<port-you-pick-on-ec2>` will not be accessible in the public internet. In order for this step to work, you need to copy the private key for accessing  your ec2 machine into this node, and then use `chmod 400 <aws-pem-file>` to set it  to read-only. Otherwise ssh will give "PermissionError" as your pemfile is too permissive.

    ```bash
    ssh -R 0.0.0.0:<port-you-pick-on-ec2>:localhost:<jaynes.server-port> -N ec2-user@<ec2-instance-ip> -i <aws-pem-file>.pem
    ```
    
    You can use screen with the daemon option, to put this into the background
    
    ```bash
    screen -dm ssh -R ...
    ```
    
    The `-N` option in the ssh reverse tunnel prevents the command from opening up a tty session.

4. Now if you try to launch, you should see print outs in your `jaynes.server`.

## Running This Example

You need to set the following environment variables:

```bash
# ~/.bashrc

export JYNS_MANAGER_HOST=http://<your-aws-server>:<jaynes.erver-port>
export JYNS_MANAGER_HOME=/home/<your-username>
```

And then you should be able to run the `launch.py` script!!

## Important Notes

1. This manager launch mode synchronously listens to the `stdout` and `stderr` of the launched jobs, therefore if your job takes a while to finish (which is pretty much all of our jobs), the call back will be blocking. So the `jaynes.run` function under this `manager` mode is also blocking.

   For this reason, we add a timeout, so that we can launch multiple jobs one after another.

2. The manager server has an issue with too many pipe-backs hanging connected to the server instance. This is an experimental implementation.

3. SSH tunnels are typically allowed on managed clusters, but reverse tunnels are trickier. In the future, we will add jasonwebtoken (JWT) based authentication scheme to `jaynes.manager`. It will make this setup more secure.

Happy Researching! It is fun.

Ge
