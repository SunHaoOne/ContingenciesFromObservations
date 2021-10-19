# Contingencies From Observations

[https://sites.google.com/view/contingency-planning/home](https://sites.google.com/view/contingency-planning/home)

![decision_tree](https://user-images.githubusercontent.com/5475622/116763176-c8612200-a9d1-11eb-9d45-9a65b60825b2.png)

## Purposes

1. Serve as the accompanying code for ICRA 2021 paper: Contingencies from Observations.
2. A framework for running scenarios with PRECOG models in CARLA.

## 安装CARLA
> 这个比较容易一些，直接去官方下载realse版本后，再把这个送到服务器上解压。
This repository requires CARLA 0.9.8. Please navigate to carla.org to download the correct packages, or do the following:
```bash
# Downloads hosted binaries
wget https://carla-releases.s3.eu-west-3.amazonaws.com/Linux/CARLA_0.9.8.tar.gz

# Unpack CARLA 0.9.8 download
tar -xvzf CARLA_0.9.8.tar.gz -C /path/to/your/desired/carla/install
```

Once downloaded, make sure that `CARLAROOT` is set to point to your copy of CARLA:
```bash
export CARLAROOT=/path/to/your/carla/install
```

`CARLAROOT` should point to the base directory, such that the output of `ls $CARLAROOT` shows the following files:
```bash
CarlaUE4     CHANGELOG   Engine  Import           LICENSE                        PythonAPI  Tools
CarlaUE4.sh  Dockerfile  HDMaps  ImportAssets.sh  Manifest_DebugFiles_Linux.txt  README     VERSION
```

## 安装
> 这个直接安装就可以了，非常顺利
```bash
conda create -n precog python=3.6.6
conda activate precog
# make sure to source this every time after activating, and make sure $CARLAROOT is set beforehand
source precog_env.sh
pip install -r requirements.txt
```
Note that `CARLAROOT` needs to be set and `source precog_env.sh` needs to be run every time you activate the conda env in a new window/shell.

Before running any of the experiments, you need to launch the CARLA server:
```bash
cd $CARLAROOT
./CarlaUE4.sh
```

## 下载CARLA数据集
> 这个可以去官方下载，注意的是超车数据一共是200个作为训练集。

The dataset used to train the models in the paper can be downloaded [at this link](https://drive.google.com/file/d/14-o8XZtqJnRRCPqX3gz-LJuOgBORcbXT/view?usp=sharing).

## 一些关于生成数据集需要注意的点

### 1. Scenario Runner错误

- 把相关代码中的`set_timeout`的时间2s设置为10s或者20s，这样确保了可以和客户端进行通讯.

### 2. 关于需要生成数据集的长度

- 也许200个episode的长度就够用了


## 生成超车数据集的代码

Alternatively, data can be generated in CARLA via the `scenario_runner.py` script:
```bash
cd Experiment
python scenario_runner.py \
--enable-collecting \
--scenario 1 \
--location 0  
```
Episode data will be stored to Experiment/Data folder.

> 在这里我们需要右键`EXPERIMENT`并且将其设置为根目录，然后右键运行文件就可以了。
Then run:
```bash
cd Experiment
python Utils prepare_data.py
```
This will convert the episode data objects into json file per frame, and store them in Data/JSON_output folder.

## CfO model 

The CfO model/architecture code is contained in the [precog](precog) folder, and is based on the [PRECOG repository](https://github.com/nrhine1/precog) with several key differences:

1. The architecture makes use of a CNN to process the LiDAR range map for contextual input instead of a feature map (see [precog/bijection/social_convrnn.py](precog/bijection/social_convrnn.py)). 
2. The social features also include velocity and acceleration information of the agents (see [precog/bijection/social_convrnn.py](precog/bijection/social_convrnn.py)).
3. The plotting script visualizes samples in a fixed set of coordinates with LiDAR overlayed on top (see [precog/plotting/plot.py](precog/plotting/plot.py)). 

## Training the CfO model

Organize the json files into the following structure:
```md
Custom_Dataset
---train
   ---feed_Episode_1_frame_90.json
   ...
---test
   ...
---val
   ...
```

Modify relevant precog/conf files to insert correct absolute paths.
> 这里需要注意的是，一些文件少写了右侧的单引号，需要补全。
```md
Custom_Dataset.yaml
esp_infer_config.yaml
esp_train_config.yaml
shared_gpu.yaml
sgd_optimizer.yaml # set training hyperparameters
```
> 这里应该都是没问题的，注意运行前需要`source precog_env.sh`
Then run:
```bash
export CUDA_VISIBLE_DEVICES=0;
python $PRECOGROOT/precog/esp_train.py \
dataset=Custom_Dataset \
main.eager=False \
bijection.params.A=2 \
optimizer.params.plot_before_train=True \
optimizer.params.save_before_train=True
```

> 此外还应该修改一些文件和内容，如添加这个返回meta_list的类型。
>

## Evaluating a trained CfO model

To evaluate a trained model in the CARLA simulator, run:
```bash
cd Experiment
python scenario_runner.py \
--enable-inference \
--enable-control \
--enable-recording \
--checkpoint_path [absolute path to model checkpoint] \
--model_path [absolute path to model folder] \
--replan 4 \
--planner_type 0 \
--scenario 0 \
--location 0
```

A checkpoint of the model used in the paper is provided in [Model/esp_train_results](Model/esp_train_results).

The example script [test.sh](Experiment/test.sh) will run the experiments from the paper and generate a video for each one. For reference, when using a Titan RTX GPU and Intel i9-10900k CPU each episode takes approximately 10 minutes to run, and the entire script takes several hours to run to completion.


## Running the MFP baseline

Install the [MFP baseline repo](https://github.com/cpacker/multiple-futures-prediction-carla), and set `MFPROOT` to point to your copy:
```bash
export MFPROOT=/your/copy/of/mfp
```

Use the `scenario_runner_mfp.py` script to run the MFP model inside of the CARLA scenarios:
```bash
# left turn
python scenario_runner_mfp.py \
--enable-inference \
--enable-control \
--enable-recording \
--replan 4 \
--scenario 0 \
--location 0 \
--mfp_control \
--mfp_checkpoint CARLA_left_turn_scenario

# right turn
python scenario_runner_mfp.py \
--enable-inference \
--enable-control \
--enable-recording \
--replan 4 \
--scenario 2 \
--location 0 \
--mfp_control \
--mfp_checkpoint CARLA_right_turn_scenario

# overtake
python scenario_runner_mfp.py \
--enable-inference \
--enable-control \
--enable-recording \
--replan 4 \
--scenario 1 \
--location 0 \
--mfp_control \
--mfp_checkpoint CARLA_overtake_scenario
```

## Citations
To cite this work, use:
```bibtex
@inproceedings{rhinehart2021contingencies,
    title={Contingencies from Observations: Tractable Contingency Planning with Learned Behavior Models},
    author={Nicholas Rhinehart and Jeff He and Charles Packer and Matthew A. Wright and Rowan McAllister and Joseph E. Gonzalez and Sergey Levine},
    booktitle={International Conference on Robotics and Automation (ICRA)},
    organization={IEEE},
    year={2021},
}
```

## License
[MIT](https://choosealicense.com/licenses/mit/)
