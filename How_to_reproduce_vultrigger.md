How to run joern for a git repo with a diff patch:

## 0. Unpack vultrigger_datasample.tar.gz

Download vultrigger_datasample.tar.gz from https://drive.google.com/file/d/1SE8NEH9vKXQbQ8WmorTlvePieuE-NEt9/view?usp=sharing

```bash
tar xvf vultrigger_datasample.tar.gz
```

## 1. Import the docker container

```bash
docker pull hkadxqq2/vultrigger_public:latest your_image_name # please give it a name different from vultrigger:v1.0

docker run -v ./vultrigger_datasample/:/data/ -it your_image_name /bin/bash # this will create a docker image where vultrigger_datasample is mounted under /data/

docker ps -a # this will show all the docker containers, including the ID of the container you just built

```

then you will find your container SHA for your_image_name, run this every time:

```bash

docker exec -it container_sha /bin/bash # after this you should see root@SHA/

cd /home/
```

## 2 How to obtain the critical variable of the diff patch

First change `config.json`'s `all_test_code` to your repo name, e.g., :

```json
"all_test_code":{
        "all_diff_path":"/data/patch_db/data/C-Diffs/neomutt@@neomutt/",
        "all_new_path":"/data/patch_db/data/C-Non_Vulnerable_Files/neomutt@@neomutt/",
        "all_old_path":"/data/patch_db/data/C-Vulnerable_Files/neomutt@@neomutt/",
        "all_dep_path":"/data/patch_db/data/Dependency_Files/"
    }
```

Run `cv_extract.py` to obtaine the critical variable location, notice it needs to be python3

```bash
cd /home/VulTrigger/VulTrigger/
python3 cv_extract.py neomatt@@neomutt 9bfab35522301794483f8f9ed60820bdec9be59e
```
The result will be stored under `/data/patch_db/data/cv_result/neomutt@@neomutt/9bfab35522301794483f8f9ed60820bdec9be59e/step1_result.txt`, e.g., it looks like this:

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

## 3.Run the VulTrigger pipeline

After entering the docker container, there is a directory /home/VulTrigger/VulTrigger, enter the directory and run

```bash
python3 run_batch_exp.py ./all_data.csv
```

The code will run and build the cfg, pdg, slicing result, sink result, etc. 

If you see the following file generated (non-empty), it means you have successfully reproduced the code:
match_results/neomutt@@neomutt/9bfab35522301794483f8f9ed60820bdec9be59e/sink_paths_json_119.json

