How to run joern for a git repo with a diff patch:

## 1. Import the docker container

```bash
docker pull hkadxqq2/vultrigger_fork:latest your_image_name # please give it a name different from vultrigger:v1.0

docker run -it your_image_name /bin/bash # this will create a docker image 

docker ps -a # this will show all the docker containers, including the ID of the container you just built

```

then you will find your container SHA for your_image_name, run this every time:

```bash

docker exec -it container_sha /bin/bash # after this you should see root@SHA/

cd /home/
```

## 2. Store the .diff for each file under ./data/C-Diffs/neomutt/:

Before the following steps, you need to change the `all_test_code` in `config.json` to the repo name you are testing. 

```bash
cd /home/VulTrigger/VulTrigger/gitrepos
git clone https://github.com/neomutt/neomutt.git
mv neomutt neo_mutt_git
cd neo_mutt_git
mkdir /home/VulTrigger/VulTrigger/data/C-Diffs/neomutt/
git show {commit_id} > /home/VulTrigger/VulTrigger/data/C-Diffs/neomutt/diff.txt
```

then separate the diff.txt into diff based on the file. For example, for this commit: https://github.com/neomutt/neomutt/commit/9bfab35522301794483f8f9ed60820bdec9be59e, separate them into 2 files:

`./data/C-Diffs/neomutt/CVE-2018-14363/CVE-2018-14363_CWE-22_9bfab35522301794483f8f9ed60820bdec9be59e_newsrc.c_.diff`

and

`./data/C-Diffs/neomutt/CVE-2018-14363/CVE-2018-14363_CWE-22_9bfab35522301794483f8f9ed60820bdec9be59e_pop.c_.diff`

each contains only the diff in that file.

## 2.5 How to obtain the critical variable of the diff patch

First change `config.json`'s `all_test_code` to your repo name, e.g., :

```json
"all_test_code":{
        "all_diff_path":"/home/VulTrigger/VulTrigger/data/C-Diffs/neomutt/",
        "all_new_path":"/home/VulTrigger/VulTrigger/data/C-Non_Vulnerable_Files/neomutt/",
        "all_old_path":"/home/VulTrigger/VulTrigger/data/C-Vulnerable_Files/neomutt/",
        "all_dep_path":"/home/VulTrigger/VulTrigger/data/Dependency_Files/"
    }
```

Run `cv_extract.py` to obtaine the critical variable location, notice it needs to be python3

```bash
cd /home/VulTrigger/VulTrigger/
python3 cv_extract.py neomatt
```
The result will be stored under `./cv_result/step1_result.txt`, e.g., it looks like this:

```bash
=======================complex type===========================
1. Patch_model: Add
Change_statement_type: Fun-Head
Line_number: new file:#75
Critical_variable: ['*id']

2. Patch_model: Add
Change_statement_type: Var-Declaration
Line_number: new file:#77
Critical_variable: ['clean']

3. Patch_model: Add
Change_statement_type: Fun-Call
Line_number: new file:#78
Critical_variable: ['clean', 'id']
```

They tell you which variable at which lines are changed, these variables can be used to find the call graph path to tell LLM agent about. You can also feed the input result of cv_extract to LLM agent, and let it pick the line of critical variable. 

## 3. Copy the repo into joern/testCode directory for analysis

```bash
cd /home/VulTrigger/VulTrigger/gitrepos/neomutt_git
rm -r /home/SySeVR/joern-0.3.1/testCode/* # remove the existing code
cd /home/VulTrigger/VulTrigger/ # go back to the directory
python get_depen.py neomutt # this script will copy the git repo to be under joern, and start the neo4j server, so you can start querying the joern graph using py2neo
```

## 4. Test that you can start querying the graph, which is like the go to caller/callee feature in PyCharm. You first need to check that joern server is running using:

```bash
/home/SySeVR/neo4j/bin/neo4j status
```

if it's not running, run

```bash
/home/SySeVR/neo4j/bin/neo4j start-no-wait
```

Run python:

```Python
root@2d7337084a97:/home/VulTrigger/VulTrigger# python
Python 2.7.12 (default, Dec  4 2017, 14:50:18) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> 
```

Load the database:

```Python
>>>from joern.all import JoernSteps
>>>j = JoernSteps()
>>>j.connectToDatabase()
```

Then you can start querying the call graph:

### * Get the node from a function name:

```Python
>> j.runGremlinQuery('getFunctionsByName("cache_id")')
[<Node graph=u'http://localhost:7474/db/data/' ref=u'node/455507' labels=set([]) properties={u'type': u'Function', u'name': u'cache_id', u'location': u'75:0:1945:2132'}>]
```

### * Get the information of a function based on ID:

```Python
>>j.runGremlinQuery("g.v('23765')")
<Node graph=u'http://localhost:7474/db/data/' ref=u'node/23765' labels=set([]) properties={u'type': u'Function', u'name': u'add_folder', u'location': u'59:0:1739:4282'}>
```

### * Get the callers of a function:

```Python
>>> j.runGremlinQuery("getCallsTo('cache_id')")
[<Node graph=u'http://localhost:7474/db/data/' ref=u'node/456281' labels=set([]) properties={u'childNum': u'0', u'code': u'cache_id ( id )', u'type': u'CallExpression', u'functionId': 456263}>, <Node graph=u'http://localhost:7474/db/data/' ref=u'node/456738' labels=set([]) properties={u'childNum': u'0', u'code': u'cache_id ( ctx -> hdrs [ i ] -> data )', u'type': u'CallExpression', u'functionId': 456557}>, <Node graph=u'http://localhost:7474/db/data/' ref=u'node/458302' labels=set([]) properties={u'childNum': u'0', u'code': u'cache_id ( h -> data )', u'type': u'CallExpression', u'functionId': 458023}>, <Node graph=u'http://localhost:7474/db/data/' ref=u'node/458497' labels=set([]) properties={u'childNum': u'0', u'code': u'cache_id ( h -> data )', u'type': u'CallExpression', u'functionId': 458023}>, <Node graph=u'http://localhost:7474/db/data/' ref=u'node/458687' labels=set([]) properties={u'childNum': u'0', u'code': u'cache_id ( h -> data )', u'type': u'CallExpression', u'functionId': 458023}>, <Node graph=u'http://localhost:7474/db/data/' ref=u'node/459058' labels=set([]) properties={u'childNum': u'0', u'code': u'cache_id ( ctx -> hdrs [ i ] -> data )', u'type': u'CallExpression', u'functionId': 458870}>]
```

### * Get the callees of a function:

```Python
>>j.runGremlinQuery("queryNodeIndex('type:Callee AND functionId: 455507')")
>>[<Node graph=u'http://localhost:7474/db/data/' ref=u'node/455524' labels=set([]) properties={u'childNum': u'0', u'code': u'mutt_file_sanitize_filename', u'type': u'Callee', u'functionId': 455507}>, <Node graph=u'http://localhost:7474/db/data/' ref=u'node/455537' labels=set([]) properties={u'childNum': u'0', u'code': u'mutt_str_strfcpy', u'type': u'Callee', u'functionId': 455507}>]
```

### * Get the file name a function is in:

```Python
>>j.runGremlinQuery("g.v('455507').in('IS_FILE_OF').filepath")
>>[u'testCode/pop.c']
```

## 5. How to run python3.9 inside of the container

The container has a miniconda Python 3.9 environment, you can activate it under the container using the following command (so you can run joern and the notebook under the same container). I have not yet figured out how to run Jupyter notebook inside of the remote running docker container, but Python works. Since python works, you can choose to clone the oneday_align project to be inside of Docker. Later we will do so and change the call graph part (e.g., mutt_bcache_del (bcache.c: 256) <- cache_id (pop.c:75) <- pop_fetch_headers (pop.c: 307) <- pop_open_mailbox (pop.c: 478), pop_check_mailbox (pop.c: 852)) from manually obtained by the annotator to automatically obtained by LLM agents. 

```bash
source ~/.bashrc
source activate base
source activate py39
```
## 6. How to run docker to access an outside directory (ie mount -v)

```bash
docker run -v your_local_directory:docker_directory -it your_image /bin/bash
```
The outside directory can be found under docker_directory inside of the container

## 7. How to run remote container using VSCode:

First, install the 'Dev Containers' plugin on VSCode. 

First make sure your container is running, then in VSCode remote ssh connection, type command+shift+P (command pallette) -> Dev Containers: Attach to a running container
