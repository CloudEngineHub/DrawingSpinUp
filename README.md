# DrawingSpinUp

Official code for DrawingSpinUp

## Environment Setup

Hardware: 
  - All experiments are run on a single RTX 2080Ti GPU.

Setup environment:
  - Python 3.8.0
  - PyTorch 1.13.1
  - Cuda Toolkit 11.6
  - Ubuntu 18.04

Install the required packages:

```sh
conda create -n drawingspinup python=3.8
conda activate drawingspinup
pip install torch==1.13.1+cu116 torchvision==0.14.1+cu116 torchaudio==0.13.1 --extra-index-url https://download.pytorch.org/whl/cu116
pip install -r requirements.txt
# tiny-cuda-nn
pip install git+https://github.com/NVlabs/tiny-cuda-nn/#subdirectory=bindings/torch
# python-mesh-raycast
git clone https://github.com/cprogrammer1994/python-mesh-raycast
cd python-mesh-raycast
python setup.py develop
```

Clone this repository and download our processed character drawings from [preprocessed.zip](https://portland-my.sharepoint.com/:u:/g/personal/jzhou67-c_my_cityu_edu_hk/EWi-CdpGraRMhbqvc7Fq9k0BulcK2or_9fjaEuWVAi97Dw?e=Sj018E) (a tiny subset of [Amateur Drawings Dataset](https://github.com/facebookresearch/AnimatedDrawings)). Of course you can prepare your own image: a 512x512 character drawing 'texture.png' with its foreground mask 'mask.png'.

```sh
git clone https://github.com/LordLiang/DrawingSpinUp.git
cd DrawingSpinUp
# download blender for frame rendering
wget https://download.blender.org/release/Blender3.3/blender-3.3.1-linux-x64.tar.xz
tar -xvf blender-3.3.1-linux-x64.tar.xz
# install trimesh for blender's python
wget https://bootstrap.pypa.io/get-pip.py
./blender-3.3.1-linux-x64/3.3/python/bin/python3.10 get-pip.py
./blender-3.3.1-linux-x64/3.3/python/bin/python3.10 -m pip install trimesh
cd dataset/AnimatedDrawings
# download preprocessed.zip and put it here
unzip preprocessed.zip
cd ../..
```

## Step-1: Contour Removal

### Inference

We use [FFC-ResNet](https://github.com/advimman/lama) as the backbone to predict the contour region of a given character drawing. 
For model training, you can refer to the original repo.
For training image rendering, see [1_lama_contour_remover/bicar_render_codes](1_lama_contour_remover/bicar_render_codes) which are borrowed from [Wonder3D](https://github.com/xxlong0/Wonder3D/tree/main/render_codes).
Here we focus on inference. Download our pretrained contour removal models from [experiments.zip](https://portland-my.sharepoint.com/:u:/g/personal/jzhou67-c_my_cityu_edu_hk/Ed6BaAAWgIhGqIMjaju_v4kB_K-DIFGu1bQ7zM3CbQMrTw?e=KaltGi).

```sh
cd 1_lama_contour_remover
# download experiments.zip and put it here
unzip experiments.zip
python predict.py
cd ..
```

## Step-2: Textured Character Generation

Firstly please download the pretrained [isnet](https://xuebinqin.github.io/dis/index.html) model ([isnet_dis.onnx](https://huggingface.co/stoned0651/isnet_dis.onnx/resolve/main/isnet_dis.onnx)) for background removal of generated multi-view images.

```sh
cd 2_charactor_reconstructor
mkdir dis_pretrained
cd dis_pretrained
wget https://huggingface.co/stoned0651/isnet_dis.onnx/resolve/main/isnet_dis.onnx
cd ..
```

Then let us take the image with the uid *0dd66be9d0534b93a092d8c4c4dfd30a* as an example. You can try your own image.

```sh
# multi-view image generation
python mv.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a
# textured character reconstruction
python recon.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a
# if shape thinning is needed, add '--thin' (optional)
python recon.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a --thin
cd ..
```

## Step-3: Stylized Contour Restoration

### Rigging & Retargeting

Once we get the textured character, we use [Mixamo](https://www.mixamo.com) to rig it automatically. Then we can directly retarget a Mixamo motion onto the rigged character online. We can also use [rokoko-studio-live-blender](https://github.com/Rokoko/rokoko-studio-live-blender) to retarget a 3D motion (e.g., *.bvh, *.fbx) onto the rigged character offline. 

### Frame Rendering

For convenience, here we offer an example [0dd66be9d0534b93a092d8c4c4dfd30a.zip]() and you can download it. 
The 'animation/fbx_files/rest_pose.fbx' is the rigged character generated by Mixamo.
The 'animation/fbx_files/dab.fbx' and 'animation/fbx_files/jumping.fbx' are two retargeted animation files. 
Then we render frames by running the scripts:

```sh
cd dataset/AnimateDrawings/preprocessed
# download 0dd66be9d0534b93a092d8c4c4dfd30a.zip and put it here
unzip 0dd66be9d0534b93a092d8c4c4dfd30a.zip
cd ../../..
cd 3_1_animation_render
# render keyframe pair for training
python run_render.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a --keyframe
# render frames for inference
python run_render.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a
cd ..
```

Then you can get the rendered frames in 'animation/blender_render' folder. The frames in 'keyframe' are used for training and the frames in 'dab' and 'jumping' are used for inference.

```sh
dataset
  └── AnimateDrawings
      └── preprocessed
          └── 0dd66be9d0534b93a092d8c4c4dfd30a
              ├── animation
              │   ├── fbx_files
              │   │   ├── rest_pose.fbx
              │   │   ├── dab.fbx
              │   │   └── jumping.fbx
              │   └── blender_render
              │       ├── keyframe
              │       │   ├── color
              │       │   ├── pos
              │       │   └── edge
              │       ├── dab
              │       │   ├── color
              │       │   ├── pos
              │       │   └── edge
              │       └── jumping
              │           ├── color
              │           ├── pos
              │           └── edge
              ├── mesh
              │   └── it3000-mc512-50000_cut_simpl_thin_shear.obj
              └── char
                  ├── texture.png
                  └── mask.png

```

### Training & Inference

We need to train a model for each sample. Once trained, the model can be applied directly to any new animation frames without further training.

```sh
cd 3_2_style_translator
python train.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a --pos --edge
python inference.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a --pos --edge
cd ..
```
If you want to generate GIF file, you can run the script:
```sh
cd 3_1_animation_render
python gif_writer.py --uid 0dd66be9d0534b93a092d8c4c4dfd30a
cd ..
```


