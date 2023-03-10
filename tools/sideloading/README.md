## Overview

**Sideloading** tool is a sample solution to add side-loading capability to your Panorama application. With this solution, you can modify script files & model files, transfer them to the application code container on the device, and run the updated scripts without re-deploying the entire application. It gives you quicker development iteration experience.

This solution comes with 1) sideloading agent module you can import in your Panorama application, 2) sideloading commandline interface you can use on your development host.

## Features

* Automatically identify updated files and transfer them. (**"sync" command**)
* Run the updated scripts and models as a process. (**"run-app" command**)
* Kill the process. (**"kill-app" command**)
* Easily switch the application from side-loading enabled development mode to production mode, with single line of code change.


## Security notice

* > We do not recommend doing this on a production system because this is a development feature that has not been AWS Security validated for use in production systems.
* > Opening a port to create a webserver could but doesn't necessarily create a security vulnerability and you will be relying on the security of the downloaded environment to protect your development environment.


## Security best practices

* > This sideloading solution uses certificate files and private key files for mutual authentication. Please keep the them secure, use them only for sideloading to your device, and don't share them widely such as adding those files in the version control system, or sharing files from public S3 bucket.
* > When creating certificate & private key files, use the minimum required expiry at longest 30 days. As needed, please implement periodic rotation of the certificate and key files so that they don't expire.
* > When you deploy your application to Production environment, follow the steps in [How to switch to Production mode](#how-to-switch-to-production-mode) in this docment.


## How to use

1. Open a terminal window on your development host. Under the top directory of your application (where assets / graphs / packages directories exist), run the "install.py" script to generate and install necessary files.

    ``` bash
    python3 aws-panorama-samples/tools/sideloading/install.py --expiry 30 --app-src-dir ./packages/123456789012-code-1.0/src
    ```

    This script creates / installs following files.

    * `sideloading_agent.py` under code package src directory.
    * `sideloading_server.cert.pem`, `sideloading_server.key.pem`, `sideloading_client.cert.pem` under code package src directory.
    * `sideloading_client.cert.pem`, `sideloading_client.key.pem`, `sideloading_server.cert.pem` under current directory.

    **Note:** Please don't include these `*.pem` files in your source code version control system. In case of Git, please edit the .gitignore file to ignore these files.

1. Create two Python scripts, 1) parent process which runs sideloading agent thread, 2) child process which runs computer vision tasks. You can decide the names of those scripts, but in the following example, we use `boot.py` as the parent process, and `cv_app.py` as the child process.

    1. Under the code package src directory, create a Python script (`boot.py`) for the parent process which runs the sideloading agent. Also modify the descriptor.json of the code asset with the created script file name. 

        boot.py :
        ``` python
        import sideloading_agent

        sideloading_agent.run(
            entrypoint_filename = "cv_app.py",
            enable_sideloading = True,
            run_app_immediately = False,
            port = 8123,
        )
        ```

        descriptor.json :
        ```json
        {
            "runtimeDescriptor": {
                "envelopeVersion": "2021-01-01"
                "entry": {
                    "name": "/panorama/boot.py",
                    "path": "python3"
                },
            }
        }
        ```

        **Note:** Please note that you need to specify the file name of the parent process in the code descriptor file.

    1. Under the code package src directory, create a Python script (`cv_app.py`) for the child process which runs the computer vision tasks. The file name has to be same as the `entrypoint_filename` argument you specified in the `boot.py`.

        cv_app.py :
        ``` python
        import panoramasdk

        class Application(panoramasdk.node):
            def __init__(self):
                ... snip ...
        ```

1. Enable inbound networking by modifying JSON files as follows.
    - Add "inboundPorts" element at the following place in the code package.json.
        ``` json
        {
            "nodePackage": {
                
                ... snip ...
                
                "interfaces": [
                    {

                        ... snip ...

                        "network": {
                            "inboundPorts": [
                                {
                                    "port": 8123,
                                    "description": "sideloading"
                                }
                            ]
                        }
                    }
                ]
            }
        }
        ```

    - Add "networkRoutingRules" element at the following place in manifest file. 
        ``` json
        {
            "nodeGraph": {

                ... snip ...

                "networkRoutingRules": [
                    {
                        "node": "code_node",
                        "containerPort": 8123,
                        "hostPort": 8123,
                        "decorator": {
                            "title": "Port for sideloading",
                            "description": "Port for sideloading."
                        }
                    }
                ]
            }
        }
        ```
        **Note:** Please make sure you use right code node name. In this example, "code_node" is specified as the code node name, but it can be different in your manifest file.

1. With regular panorama application building & packaging steps, build and package the side liading enabled application. (`panorama-cli build-container` command, `panorama-cli package-application` command).

1. Deploy the application onto your device.
    * Option 1 (basic) : Deploy using Management Console UI.
        1. At the "Configure" step in the deployment wizard, click the "Configure application" button. ![](images/configure_inbound_port_1.png)
        1. Choose "Inbound ports" tab. ![](images/configure_inbound_port_2.png)
        1. Enable "Open port", configure "Device port" (default: 8123), and "Save" ![](images/configure_inbound_port_3.png)
    * Option 2 (advanced) : Deploy using AWSCLI or API.
        1. Include the "networkRoutingRules" element in the override manifest file as well.
            ``` json
            {
                "nodeGraphOverrides":{

                    ... snip ...

                    "networkRoutingRules":[
                        {
                            "node": "code_node",
                            "containerPort": 8123,
                            "hostPort": 8123
                        }
                    ]
                }
            }
            ```
            **Note:** Please make sure you use right code node name. In this example, "code_node" is specified as the code node name, but it can be different in your manifest file.

        1. Deploy the application programmatically using boto3's create_application_instance() API or AWSCLI's "aws panorama create-application-instance" command. Specify contents of graph.json as the manifest payload, and contents of override.json as the override manifest payload. (See also : [Automate application deployment](https://docs.aws.amazon.com/panorama/latest/dev/api-deploy.html)).

1. Confirm the completion of deployment.
    * **Note** : After the deployment completed, you will see `Error` or `Not available` status. You can safely ignore these errors. It just means the application hasn't started regular computer vision process yet.

1. Check "console_output" log stream on CloudWatch Logs, and confirm "SideloadingAgent started" message.

1. Identify the IP address of your appliance device. You can use the Management Console UI or "aws panorama describe-device" command.

1. Under the top directory of your application (where assets / graphs / packages directories exist), hit "`sideloading_cli.py sync`" command to transfer updated files. Following is an example:

    ```bash
    $ python3 aws-panorama-samples/tools/sideloading/sideloading_cli.py sync --addr 10.0.0.245 --port 8123 --src-top packages/123456789012-mycode-1.0/src
    ```

    **Note:** This is an example. Please replace script path, IP address of the device, port number, and path to the "src" directory according to your environment.

    **Note:** By default, sideloading_cli.py tries to find PEM files from current directory. Alternatively you can specify the path with `--cert_key_dir` option.

1. Under the top directory of your application (where assets / graphs / packages directory exist), hit "`sideloading_cli.py run-app`" command to run application with transfered files. Following is an example:

    ```bash
    $ python3 aws-panorama-samples/tools/sideloading/sideloading_cli.py run-app --addr 10.0.0.245 --port 8123
    ```

1. Reboot the device, and repeat "sync" and "run-app".


## How to switch to Production mode

When you completed the development and are ready to deploy to production system, please take following steps.

1. Modify your Panorama application entry point script (e.g. "boot.py") as follows:
    ``` python
    import sideloading_agent

    sideloading_agent.run(
        entrypoint_filename = "cv_app.py",
        enable_sideloading = False,
        run_app_immediately = True,
    )
    ```
1. Remove `sideloading_*.pem` files from the code package src directory.
1. With regular panorama application building & packaging steps, build and deploy the application onto your device without enabling inbound networking port. (keep the "Open port" checkbox disabled when deploying.)


## How it works

1. Transfered files are stored under `/opt/aws/panorama/storage/sideloaded/`. This is a different directory from Panorama application's standard application directory - `/panorama/`.

1. `run-app` command tries to find application script file in the following priority order. The first script found is executed as a child process.

    1. `/opt/aws/panorama/storage/sideloaded/{entrypoint_filename}`
    1. `/panorama/{entrypoint_filename}`

    **Note:** `entrypoint_filename` is the file name specified as the argument for `sideloading_agent.run()`. The extension of the file name has to be either `.py` or `.sh`.

    ![](images/directory.png)


1. When running the child process, working directory of the process is set at the directory of the script found - `/opt/aws/panorama/storage/sideloaded/` or `/panorama/`. This is different from Panorama applications' standard behavior where working directory is set at `"/"` (root directory). 
    
    * In order for your application to accesses sideloaded files (model data, configuration file, etc), you can use relative paths such as "./model.bin", "./config.json". If you use absolute paths such as "/panorama/config.json", it doesn't refer to sideloaded files.


## Limitations

* Your Panorama appliance device and your development host machine have to be in a same network, as sideloading_cli.py needs to access your Panorama appliance device.
* panoramasdk.node instance is intended to be created just once within a lifetime of Panorama application. If you use run-app command and kill-app command and create `panoramasdk.node` instances multiple times as a result, the behavior of panoramasdk.node APIs is indeterministic. Please reboot the device in this case.
* You cannot sideload native libraries. All the native libaries have to be included in the code container image before deployment.
