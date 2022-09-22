# Container

- the container only lives as long as the process inside it is alive

**How to run a container ?**

`docker run ubuntu`

- docker starts a container using ubuntu image that has `CMD ["bash"]` as "running process"
- bash is a shell that listens for input from terminal , in our case bash doenst find a terminal and it exits

**How to specify a different command ?**

`docker run ubuntu sleep 5`

- container will sleep 5 sec and will exit

**How to specify a different command permanently?**

- create a new Dockerfile

~~~
FROM ubuntu
CMD sleep 5
or 
CMD ["sleep", "5"] # command and arg should be separate elements of the list
~~~

- build the image

`docker build -t myubuntu .`

- run container

`docker run myubuntu`

- container will sleep 5 sec and will exit

**What if you want to sleep 10 ?**

`docker run myubuntu sleep 10`

**Better option ?**

- create Dockerfile and use ENTRYPOINT

~~~
FROM ubuntu
ENTRYPOINT ["sleep"]
~~~

- run the container by passing only the arg (10) to the starting process (ENTRYPOINT - sleep in our case)
- 
`docker run myubuntu 10`

* if you dont specify 10 from command will errror, solution is next

------------------------

CMD        - command line parameter will get replaced entirely

ENTRYPOINT - command line parameter will get appended

------------------------

- the best of both worlds:

~~~
FROM ubuntu
ENTRYPOINT ["sleep"]
CMD ["5"]
~~~

- if run `docker run myubuntu `   - sleep 5
- if run `docker run myubuntu 10` - sleep 10

- ENTRYPOINT doesnt change , but the argument changes

- to overwrite entrypoint

`docker run --entrypoint sleep2.0 myubuntu 10` (assuming sleep2.0 is valid command)

- this will execute `sleep2.0 10` and exit

--------------------------

# Pod
~~~
cat ubuntu-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: ubuntu
spec:
  containers:
  - name: ubuntu
    image: myubuntu
    command: ["sleep2.0"]
    args: ["10"]
~~~

~~~
Container            -------  Pod

ENTRYPOINT ["sleep"] -------  command: ["sleep2.0"]    (overwrites)
CMD        ["5"]     -------  args:    ["10"]          (overwrites)
~~~
