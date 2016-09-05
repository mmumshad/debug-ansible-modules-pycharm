### Introduction
I have been developing custom [Ansible](https://www.ansible.com/) modules on my windows system using [JetBrains PyCharm IDE](https://www.jetbrains.com/pycharm/). Below I describe some challenges and best practices on the development and how to debug remotely. 

For information about developing custom modules using Ansible check [here](http://docs.ansible.com/ansible/developing_modules.html).

## Challenge
Though Ansible supports configuring Windows systems, Ansible itself [cannot](http://docs.ansible.com/ansible/intro_windows.html#reminder-you-must-have-a-linux-control-machine) be run on a windows control machine. If you are developing custom Ansible modules on a Windows Machine you might find it difficult to debug. This can be difficult if you are used to debugging in the IDE. The [test-module](https://github.com/ansible/ansible/blob/devel/hacking/test-module) offers some debug options if you are good with debugging in the Linux console. 

In this article I explain how to debug using the IDE debugger on a windows system by running the module remotely on a Ansible controller Linux system.

### Development Environment Setup and Pre-requisites
Below is my development environment:
* Development system - Windows 7
* Development IDE - [JetBrains PyCharm IDE](https://www.jetbrains.com/pycharm/) Professional Edition *
* Ansible Controller - OpenSuse Linux Machine with Ansible installed. Must have network connectivity with the Development Windows System.
* Must have the Ansible devel package downloaded on the Linux Controller Machine to test custom modules - [test-module](https://github.com/ansible/ansible/blob/devel/hacking/test-module). For instructions on setting this up, please see [Installation](http://docs.ansible.com/ansible/intro_installation.html).

>\* Must have a [JetBrains PyCharm Professional Edition](https://www.jetbrains.com/pycharm/download) for remote debugging features. Community edition [does not](https://www.jetbrains.com/pycharm/features/) support remote debugging. If you have not purchased already You can get a free trial of Professional Edition for 30 days.

### Develop Custom Ansible Module

Create a new project in JetBrains PyCharm and create a new file in the project named - 'OSCheckModule.py'

If you have never developed a custom Ansible module, start [here](http://docs.ansible.com/ansible/developing_modules.html).
Start from the Common base template and build your module. Here I have a custom module called **OSCheckModule** that checks the OS flavor against a given input :

```python
#!/usr/bin/python

try:
    import json
except ImportError:
    import simplejson as json

import os

DOCUMENTATION = '''
---
module: OSCheckModule
short_description: A custom module to check OS of target machine.
description:
        - A custom module to check OS of target machine.
options:
  os_name:
    description:
      - Name of OS to check
    required: true
'''

EXAMPLES = '''
# Example
module_name:
    os_name: Linux
'''


def main():
    module = AnsibleModule(
        argument_spec=dict(
            os_name=dict(required=True),
        )
    )

    # Retreive parameters
    os_name = module.params['os_name']

    # Check for OS
    if os_name in os.uname():
        # Successfull Exit
        module.exit_json(changed=True, msg="OS Match")
    else:
        # Fail Exit
        module.fail_json(msg="OS Mismatch. OS =" + os.uname()[0])


from ansible.module_utils.basic import AnsibleModule

if __name__ == '__main__':
    main()
```

We are ready with our first custom module. We will now copy this fiel to our Linux Controller Machine for execution. We can use PyCharm's remote Python interpretor feature to push and run the script remotely on the Linux Controller machine without having to manually copy over. Read about it [here](https://www.jetbrains.com/help/pycharm/2016.1/configuring-remote-python-interpreters.html). Will blog about it another time.

### Executing Custom Ansible Module

I have copied my new module file to the path `/opt/ansible/custom_modules` in my Linux Ansible Controller Machine. I can now use the [test-module](https://github.com/ansible/ansible/blob/devel/hacking/test-module) to test and execute my new custom ansible module. 

Syntax:
```
/opt/ansible/ansible-devel/hacking/test-module -m <module file> -a <module arguments>
```

Example:

```
root:/opt/ansible/custom_modules # /opt/ansible/ansible-devel/hacking/test-module -m OSCheckModule.py -a os_name=Linux
* including generated source, if any, saving to: /root/.ansible_module_generated
* this may offset any line numbers in tracebacks/debuggers!
***********************************
RAW OUTPUT

{"msg": "OS Match", "invocation": {"module_args": {"os_name": "Linux"}}, "changed": true}


***********************************
PARSED OUTPUT
{
    "changed": true,
    "invocation": {
        "module_args": {
            "os_name": "Linux"
        }
    },
    "msg": "OS Match"
}

```

Wow!! we have successfully created and tested our first Custom Ansible Module.

### Debugging Custom Ansible Module

We would now like to debug our custom ansible module using PyCharm debugger. For example we would like to set a break point at Line 46 to inspect the result of os.uname() call. We wouldwill use Python [pydevd](https://pypi.python.org/pypi/pydevd) for this purpose.

#### Install pydevd on the Linux Machine Controller

```
pip install pydevd
```

#### Setup PyCharm Debug Server
Create a new Project in PyCharm and go to **Run -> Edit Configurations..**

Click on the plus sign in top left corner and select Python Remote Debug. (You must be using PyCharm Professional Edition to have this feature. Ths is not available in PyCharm Community Edition.)

![remote debugger](https://github.com/mmumshad/debug-ansible-modules-pycharm/blob/master/screenshots/pycharm_new_remote_debugger.png)

Provide the following information :
* **Name**: Give a name to the debugger - OSCheckModule_Debugger
* **Local host name**: Enter the IP of the Windows development system. Make sure you can ping and establish connectivity to this IP from your Linux System
* **Port** - Set port to 54654
* Path mappings - Ignore
*  Leave the remaining settings to default

![remote debugger configuratin](https://github.com/mmumshad/debug-ansible-modules-pycharm/blob/master/screenshots/py_charm_debugger_new_configuration.PNG)

Start the PyCharm Debug server by clicking on the debug optionbutton in the top right corner

![debug button](https://github.com/mmumshad/debug-ansible-modules-pycharm/blob/master/screenshots/pycharm_debug_icon.PNG)

The Debug server starts and starts listening for connections

![debug server](https://github.com/mmumshad/debug-ansible-modules-pycharm/blob/master/screenshots/pycharm_debug_server.PNG)

Note down the instructions provided to configure source code:
```
Use the following code to connect to the debugger:
import pydevd
pydevd.settrace('10.123.12.214', port=54654, stdoutToServer=True, stderrToServer=True)
```

#### Configure source code:

Insert the following lines in the source code of the custom module - OSCheckModule.py

Insert import pydevd line at the top
```
import pydevd
```

Insert the below line to set breakpoint at line 46

```
pydevd.settrace('10.123.12.214', port=54654, stdoutToServer=True, stderrToServer=True)
```

#### Start debugging

Copy the source code to the linux system, if you have made the changes on your windows system. Run the module using test-module:

```
root:/opt/ansible/custom_modules # /opt/ansible/ansible-devel/hacking/test-module -m OSCheckModule.py -a os_name=Linux
```

If connectivity is established between the Linux and Windows systems, a prompt will appear in PyCharm in windows. 

![debug server prompt](https://github.com/mmumshad/debug-ansible-modules-pycharm/blob/master/screenshots/py_charm_debugger_prompt.PNG)

Select the **Download Source** option to downlaod the source code to windows machine. The source will be downloaded the breakpoint is set. You can line step through the remainder of the code and use Debug console to view variables and their values.

![debug server console](https://github.com/mmumshad/debug-ansible-modules-pycharm/blob/master/screenshots/pycharm_debug_variables.PNG)

Thank you for reading. Please feel free to comment, propose better solutions.


### Authors and Contributors
Mumshad Mannambeth (@mmumshad)

### Support or Contact
See a problem, have an issue? Raise an issue at the [github page](https://github.com/mmumshad/debug-ansible-modules-pycharm)
