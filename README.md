## EDM repo
documentation by sqa

## Requirements (By EDM)

* 64-bit Python 3.8 and PyTorch 1.12.0 (or later). 
* Python libraries: See [environment.yml](./environment.yml) for exact library dependencies. You can use the following commands with Miniconda3 to create and activate your Python environment:
  - `conda env create -f environment.yml -n edm`
  - `conda activate edm`
* Docker users:
  - Ensure you have correctly installed the [NVIDIA container runtime](https://docs.docker.com/config/containers/resource_constraints/#gpu).
  - Use the [provided Dockerfile](./Dockerfile) to build an image with the required library dependencies.

## Preparing datasets

**AFHQv2:** 

1. Open [this dropbox url](https://www.dropbox.com/s/vkzjokiwof5h8w6/afhq_v2.zip?dl=0) and download the `afhq_v2.zip` file.

2. put the `afhq_v2.zip` file in the `downloads` directory under `edm-sqa`.

3. Run

```.bash
unzip ./download/afhq_v2.zip -d ./download/afhqv2/
```

Then hopefully you will see `/train` and `/test` under `./download/afhqv2/`.

4. preprocess into 64x64 and calculate FID ref:

```.bash
python dataset_tool.py --source=downloads/afhqv2 \
    --dest=datasets/afhqv2-64x64.zip --resolution=64x64
python fid.py ref --data=datasets/afhqv2-64x64.zip --dest=fid-refs/afhqv2-64x64.npz
```

5. sanity: (which will take 16 hours on one 4090, not recommended)

```.bash
# Generate 50000 images and save them as fid-tmp/*/*.png
torchrun --standalone --nproc_per_node=1 generate.py --steps=40 --outdir=fid-tmp --seeds=0-49999 --subdirs \
    --network=https://nvlabs-fi-cdn.nvidia.com/edm/pretrained/edm-afhqv2-64x64-uncond-vp.pkl

# Calculate FID
torchrun --standalone --nproc_per_node=1 fid.py calc --images=fid-tmp \
    --ref=fid-refs/afhqv2-64x64.npz
```

6. Another option: eval the FID for two different refs (our calculated and edm provided)

```.bash
torchrun --standalone --nproc_per_node=1 fid.py sqa --edm_path=https://nvlabs-fi-cdn.nvidia.com/edm/fid-refs/afhqv2-64x64.npz \
    --sqa_ref=fid-refs/afhqv2-64x64.npz
```

the result should be about 5e-6.

**FFHQ:** 

1. follow the instructions in `https://github.com/qiaosungithub/ffhq-sqa.git`, and we get a folder `\images1024x1024`.

2. Move the folder to `downloads/ffhq/images1024x1024`.

3. preprocess into 64x64 and calculate FID ref:

```.bash
python dataset_tool.py --source=downloads/ffhq/images1024x1024 \
    --dest=datasets/ffhq-64x64.zip --resolution=64x64
python fid.py ref --data=datasets/ffhq-64x64.zip --dest=fid-refs/ffhq-64x64.npz
```

4. eval the FID for two different refs (our calculated and edm provided)

```.bash
torchrun --standalone --nproc_per_node=1 fid.py sqa --edm_path=https://nvlabs-fi-cdn.nvidia.com/edm/fid-refs/ffhq-64x64.npz \
    --sqa_ref=fid-refs/ffhq-64x64.npz
```

the result should be about 3e-6.

## Training new models

**AFHQv2 64x64:** 

run **VP**:

```.bash
torchrun --standalone --nproc_per_node=8 train.py --outdir=training-runs \
    --data=datasets/afhqv2-64x64.zip --cond=0 --arch=ddpmpp --batch=256 --cres=1,2,2,2 --lr=2e-4 --dropout=0.25 --augment=0.15
```

run **VE**:

```.bash
torchrun --standalone --nproc_per_node=8 train.py --outdir=training-runs \
    --data=datasets/afhqv2-64x64.zip --cond=0 --arch=ncsnpp --batch=256 --cres=1,2,2,2 --lr=2e-4 --dropout=0.25 --augment=0.15
```

__For evaluating FID__: in the command below, change `--network` to the path of the latest network snapshot in the training directory.

```.bash
rm -rf fid-tmp
mkdir fid-tmp

# Generate 50000 images and save them as fid-tmp/*/*.png
torchrun --standalone --nproc_per_node=1 generate.py --steps=40 --outdir=fid-tmp --seeds=0-49999 --subdirs \
    --network=network-snapshot-*.pkl

# Calculate FID
torchrun --standalone --nproc_per_node=1 fid.py calc --images=fid-tmp \
    --ref=fid-refs/afhqv2-64x64.npz
```

**FFHQ 64x64:** 

run **VP**:

```.bash
torchrun --standalone --nproc_per_node=8 train.py --outdir=training-runs \
    --data=datasets/ffhq-64x64.zip --cond=0 --arch=ddpmpp --batch=256 --cres=1,2,2,2 --lr=2e-4 --dropout=0.05 --augment=0.15
```

run **VE**:

```.bash
torchrun --standalone --nproc_per_node=8 train.py --outdir=training-runs \
    --data=datasets/ffhq-64x64.zip --cond=0 --arch=ncsnpp --batch=256 --cres=1,2,2,2 --lr=2e-4 --dropout=0.05 --augment=0.15
```

__For evaluating FID__: in the command below, change `--network` to the path of the latest network snapshot in the training directory.

```.bash
rm -rf fid-tmp
mkdir fid-tmp

# Generate 50000 images and save them as fid-tmp/*/*.png
torchrun --standalone --nproc_per_node=1 generate.py --steps=40 --outdir=fid-tmp --seeds=0-49999 --subdirs \
    --network=network-snapshot-*.pkl

# Calculate FID
torchrun --standalone --nproc_per_node=1 fid.py calc --images=fid-tmp \
    --ref=fid-refs/ffhq-64x64.npz
```

Here are documentations by edm repo:

The above example uses the default batch size of 512 images (controlled by `--batch`) that is divided evenly among 8 GPUs (controlled by `--nproc_per_node`) to yield 64 images per GPU. Training large models may run out of GPU memory; the best way to avoid this is to limit the per-GPU batch size, e.g., `--batch-gpu=32`. This employs gradient accumulation to yield the same results as using full per-GPU batches. See [`python train.py --help`](./docs/train-help.txt) for the full list of options.

The results of each training run are saved to a newly created directory, for example `training-runs/00000-cifar10-cond-ddpmpp-edm-gpus8-batch64-fp32`. The training loop exports network snapshots (`network-snapshot-*.pkl`) and training states (`training-state-*.pt`) at regular intervals (controlled by `--snap` and `--dump`). The network snapshots can be used to generate images with `generate.py`, and the training states can be used to resume the training later on (`--resume`). Other useful information is recorded in `log.txt` and `stats.jsonl`. To monitor training convergence, we recommend looking at the training loss (`"Loss/loss"` in `stats.jsonl`) as well as periodically evaluating FID for `network-snapshot-*.pkl` using `generate.py` and `fid.py`.
