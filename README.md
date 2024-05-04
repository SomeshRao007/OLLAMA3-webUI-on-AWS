# OLLAMA3-webUI-on-AWS

Fun project !!

- Everything regarding how to run Ollama 3 in AWS and how to use WEB-UI.
- currently, it has manual steps I am planning to add a script and a cloud formation (or terrafom) template because I am too lazy to repeat the same steps. 


**What are ollama? and LLAMA3?**

Let's Be clear that Ollama3 isn't directly a large language model (LLM) itself, but it's a tool that allows you to run various LLMs, including Llama3, locally on your machine.

**Ollama**

- It's a software framework that simplifies setting up and running LLMs.
- It manages aspects like downloading models, configuring hardware resources, and providing an interface to interact with the LLM.
- Ollama supports various LLMs like Llama3, Mistral, Gemma, and others.


**Llama3**

- It's an open-source LLM developed by Meta AI.
- It comes in different sizes (e.g., 8B and 70B parameters) offering a trade-off between model complexity and resource requirements.
- Llama3 excels in various benchmark tasks, demonstrating strong text generation and comprehension abilities.


Uhhh!!! Boringggggg Dude !!  I know but bear with me.


> Check List !!!
> > If you are doing this on AWS or Azure steps are pretty much the same butt, you need to request your cloud providers to grant access for GPU❤️ based instances. For this, I am using AWS --> instance g4dn.xlarge (yes that **n** is for Nvidia)  =  --> it has 4vcpus, Memory 16 GB and 16 GB VRam of Nvidia Tesla T4. 


## Steps To Set Up

1. Login into your AWS account Duhh!
2. Search for EC2 then click on launch instance.
3. Name it then select a deep learning AMI that comes with tensor flow installed I am using Amazon linux 2 deep learning AMI.

![image](https://github.com/SomeshRao007/OLLAMA3-webUI-on-AWS/assets/111784343/8d85fccc-c840-4bc5-8549-a0b1b860a4a1)

4. Select instance type g4dn.xlarge (if  you have a 7-digit bank balance then go for .2xlarge or.4xlarge. Otherwise for this .xlarge is enough).
5. Create a key pair or allocate a key pair if you have one already.
6. Create a security group and add inbound rules for SSH and two custom TCP rules for port 11434 and 8080. And if you don't give a shit about all these then allow all inbound traffic.
7. Configure approx 45 to 50 GB of EBS storage and HIT that Launch button.


that's it let's go!


## Installing and Setting up OLLAMA and models 


Installing Ollama is pretty easy, run this (for easy of permissions perform all operations on root user **sudo su**) : 

~~~
curl -fsSL https://ollama.com/install.sh | sh
~~~

Downloading the language model even easier, choose a model from their library, and the following command:

~~~
ollama pull llama3
~~~

If you want to check all the models then:

~~~
ollama list
~~~


![image](https://github.com/SomeshRao007/OLLAMA3-webUI-on-AWS/assets/111784343/7b4cedeb-966e-4eba-96df-5d462aed368e)

(I was playing with other models too that's y llama2)


**To run this :**  

~~~
ollama run --verbose llama3
~~~

![Screenshot 2024-05-04 185837](https://github.com/SomeshRao007/OLLAMA3-webUI-on-AWS/assets/111784343/0fa9d909-d13e-4071-ae96-f8777c4e2ae6)

To exit just type /bye 



## How to expose this outside of localhost 


Everything up until now was through CLI and it is boring! To spice things up.

It is receiving traffic from the local host i.e it cannot connect to an external machine to check that:

``` netsat -a | grep 11434 ```

(11434 is default port for ollama.)


![image](https://github.com/SomeshRao007/OLLAMA3-webUI-on-AWS/assets/111784343/125f8500-f2b6-41cf-8039-385bbad8cce4)


To log in from other machines let's allow Ollama to receive traffic from anywhere. 

```
systemctl edit ollama
```

this will open **ollama.service** file, add this lines:

```

[Service]
Environment="OLLAMA_HOST=0.0.0.0"

```

ctrl + y and Enter (to save)
ctrl + x (to exit)

now we need to restart daemon and ollama

```
systemctl daemon-reload

systemctl restart ollama\

```

![image](https://github.com/SomeshRao007/OLLAMA3-webUI-on-AWS/assets/111784343/eff2b56b-7ca6-4ed2-91e4-2f66a9be37d4)


if you notice something changed localhost:11434  --> [::]:11434   shows now our model can now receive traffic from other machines. 

> Experience talk!!
>> I had been trying repeatedly to run the Ollama-webui Docker container using the command found on its GitHub page, but it kept failing to connect to the Ollama API server on my Linux host operating system. I realized the issue stemmed from the fact that the Ollama-WebUI server was trying to connect to Ollama on http://localhost:11434 from within the Docker container, assuming that Ollama was also installed inside the container itself. However, this wasn't the case, as Ollama was actually installed on the host OS. To resolve this, I needed to find a way to instruct Docker to redirect the connection from the container to the host OS. After some research, I found out how to write the command.


```

docker run -d --network="host" -v ollama-webui:/app/backend/data -e OLLAMA_API_BASE_URL=http://<your instance public IP>:11434/api --name ollama-webui ghcr.io/ollama-webui/ollama-webui:main

```

**VOLLAA!!! checkout publicIP:8080 for Ollama webUI**


## lazy stuff

Now if you are like me and you do not want to run that command every time you use it then do this!

To run the provided Docker command automatically upon restarting your PC, we can create a systemd service. 

1. Create a new systemd service file with a .service extension, **ollama-webui.service**, in the ``` /etc/systemd/system/ ``` directory.
2. Open the file with a text editor, and add the following lines:

```
[Unit]
Description=Ollama WebUI Docker Container
Requires=docker.service
After=docker.service

[Service]
ExecStart=/usr/bin/docker run -d --network="host" -v ollama-webui:/app/backend/data -e OLLAMA_API_BASE_URL=http://$(curl http://169.254.169.254/latest/meta-data/public-ipv4):11434/api --name ollama-webui ghcr.io/ollama-webui/ollama-webui:main
ExecStop=/usr/bin/docker stop ollama-webui
ExecStopPost=/usr/bin/docker rm ollama-webui
Restart=always

[Install]
WantedBy=multi-user.target

```

>> This command uses curl to retrieve the public IP address from the EC2 instance metadata service (http://169.254.169.254/latest/meta-data/public-ipv4).

3. Save the file and exit the text editor.
4. Reload the systemd daemon and start our ollama service:

``` systemctl daemon-reload ```

``` systemctl enable ollama-webui.service ```

``` systemctl start ollama-webui.service ``` 


![image](https://github.com/SomeshRao007/OLLAMA3-webUI-on-AWS/assets/111784343/48f57c8d-03cc-4696-9483-c9f79d13e633)


ENJOY !! the ollama-webui Docker container will start automatically upon restarting your PC. The systemd service will ensure that the container is running and will restart it if it stops unexpectedly. 

> Thank me later 😜!!



