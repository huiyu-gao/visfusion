# VisFusion: Visibility-aware Online 3D Scene Reconstruction from Videos (CVPR 2023)

### [Project Page](https://huiyu-gao.github.io/visfusion) | [Paper](https://arxiv.org/abs/2304.10687) | [Supplementary](https://huiyu-gao.github.io/visfusion/resources/VisFusion_Supp.pdf)

<br/>

<center><img src="media/scene0785_00.gif" alt=""></center>

<br/>

## Installation
```shell
sudo apt install libsparsehash-dev
conda env create -f environment.yaml
conda activate neucon
```

## ScanNet Dataset
We use the same input data structure as [NeuralRecon](https://zju3dv.github.io/neuralrecon/). You could download and extract ScanNet v2 dataset by following the instructions provided at http://www.scan-net.org/ or the [scannet_wrangling_scripts](https://github.com/nianticlabs/simplerecon/tree/main/data_scripts/scannet_wrangling_scripts) provided by [SimpleRecon](https://nianticlabs.github.io/simplerecon/). 

Expected directory structure of ScanNet:
```
DATAROOT
└───scannet
│   └───scans
│   |   └───scene0000_00
│   |       └───color
│   |       │   │   0.jpg
│   |       │   │   1.jpg
│   |       │   │   ...
│   |       │   ...
│   └───scans_test
│   |   └───scene0707_00
│   |       └───color
│   |       │   │   0.jpg
│   |       │   │   1.jpg
│   |       │   │   ...
│   |       │   ...
|   └───scannetv2_test.txt
|   └───scannetv2_train.txt
|   └───scannetv2_val.txt
```

Then generate the input fragments and the ground truth TSDFs for the training/val data split by
```bash
python tools/tsdf_fusion/generate_gt.py --data_path PATH_TO_SCANNET --save_name all_tsdf_9 --window_size 9
```
and for the test split by
```bash
python tools/tsdf_fusion/generate_gt.py --test --data_path PATH_TO_SCANNET --save_name all_tsdf_9 --window_size 9
```

## Example data
We provide an example ScanNet scene (scene0785_00) to quickly try out the code. Download it [from here](https://drive.google.com/file/d/1bEj6CVFrHZAOiY4Ir1oDvsrq4bXqga0l/view?usp=sharing) and unzip it into the main directory of the project code.

The reconstructed meshes will be saved to `PROJECT_PATH/results`.
```bash
python main.py --cfg ./config/test.yaml \
                SCENE scene0785_00 \ 
                TEST.PATH ./example_data/ScanNet \ 
                LOGDIR: ./checkpoints \ 
                LOADCKPT pretrained/model_000049.ckpt
```

By default, it will output double layer meshes (for NeuralRecon's evaluation). Set `MODEL.SINGLE_LAYER_MESH=True` to directly output single layer meshes for TransformerFusion's evaluation.
```bash
python main.py --cfg ./config/test.yaml \
                SCENE scene0785_00 \ 
                TEST.PATH ./example_data/ScanNet \ 
                LOGDIR: ./checkpoints \ 
                LOADCKPT pretrained/model_000049.ckpt \ 
                MODEL.SINGLE_LAYER_MESH True
```


## Training
Change `TRAIN.PATH` to your own data path in `config/train.yaml` and start training by running `./train.sh`.

train.sh:
```bash
#!/usr/bin/env bash
export CUDA_VISIBLE_DEVICES=0

python main.py --cfg ./config/train.yaml TRAIN.EPOCHS 20 MODEL.FUSION.FUSION_ON False
python main.py --cfg ./config/train.yaml TRAIN.EPOCHS 41 TRAIN.FINETUNE_LAYER None MODEL.PASS_LAYERS 2
python main.py --cfg ./config/train.yaml TRAIN.EPOCHS 44 TRAIN.FINETUNE_LAYER 0 MODEL.PASS_LAYERS 0
python main.py --cfg ./config/train.yaml TRAIN.EPOCHS 47 TRAIN.FINETUNE_LAYER 1 MODEL.PASS_LAYERS 1
python main.py --cfg ./config/train.yaml TRAIN.EPOCHS 50 TRAIN.FINETUNE_LAYER 2 MODEL.PASS_LAYERS 2
```

The training is seperated to five phases:

-  Phase 1 (epoch 1 - 20), training single fragments.
`MODEL.FUSION.FUSION_ON=False`

- Phase 2 (epoch 21 - 41), training the whole model with `GRUFusion`.
`MODEL.FUSION.FUSION_ON=True`

- Phase 3 (epoch 42 - 44), finetune the first layer with `GRUFusion`.
`MODEL.FUSION.FUSION_ON=True`, `TRAIN.FINETUNE_LAYER=0`, `MODEL.PASS_LAYERS=0`

- Phase 4 (epoch 45 - 47), finetune the second layer with `GRUFusion`.
`MODEL.FUSION.FUSION_ON=True`, `TRAIN.FINETUNE_LAYER=1`, `MODEL.PASS_LAYERS=1`

- Phase 5 (epoch 48 - 50), finetune the third layer with `GRUFusion`.
`MODEL.FUSION.FUSION_ON=True`, `TRAIN.FINETUNE_LAYER=2`, `MODEL.PASS_LAYERS=2`


## Test
Change `TEST.PATH` to your own data path in `config/test.yaml` and start testing by running

```bash
python main.py --cfg ./config/test.yaml
```

## Evaluation
We use NeuralRecon's evaluation for our main results.
```bash
python tools/evaluation.py --model ./results/scene_scannet_checkpoints_fusion_eval_49 --n_proc 16
```
You could print previous evaluation results by
```bash
python tools/visualize_metrics.py --model ./results/scene_scannet_checkpoints_fusion_eval_49
```
Here is the 3D metrics on ScanNet generated by the provided checkpoint using NeuralRecon's evaluation:
| Acc ↓| Comp ↓ | Chamfer ↓| Prec ↑ | Recall ↑ | F-Score↑ |   
|----------|----------|----------|----------|----------|----------|
| 5.6 | 10.0 | 7.80 | 0.694 | 0.537 | 0.604 |

and using [TransformerFusion's evaluation](https://github.com/AljazBozic/TransformerFusion/blob/main/src/evaluation/eval.py) (set `MODEL.SINGLE_LAYER_MESH=True` to output single layer mesh directly):
| Acc ↓| Comp ↓ | Chamfer ↓| Prec ↑ | Recall ↑ | F-Score↑ |   
|----------|----------|----------|----------|----------|----------|
| 4.10 | 8.66 | 6.38 | 0.757 | 0.588 | 0.660 |


## ARKit data
To try with your own data captured from ARKit, please refer to NeuralRecon's [DEMO.md](https://github.com/zju3dv/NeuralRecon/blob/master/DEMO.md) for more details.
```bash
python test_scene.py --cfg ./config/test_scene.yaml \ 
                     DATASET ARKit \ 
                     TEST.PATH ./example_data/ARKit_scan \ 
                     LOADCKPT pretrained/model_000049.ckpt
```


## Citation
If you find our work useful in your research please consider citing our paper:


```bibtex
@article{gao2023visfusion,
  title={{VisFusion: Visibility-aware Online 3D Scene Reconstruction from Videos},
  author={Gao, Huiyu and Mao, Wei and Liu, Miaomiao},
  booktitle={CVPR},
  year={2023}
}
```


## Acknowledgment
This repository is partly based on the repo [NeuralRecon](https://github.com/zju3dv/NeuralRecon). Many thanks to Jiaming Sun for the great code!
