# Head2Head: Video-based Neural Head Synthesis

## Installation

Clone the repository
```bash
git clone https://github.com/michaildoukas/head2head.git
cd head2head
```

We provide two alternatives for installing head2head required packages:
- Build a Docker image (Recommended, requires sudo privileges)
- Create a Conda environment (Requires python 3.7, CUDA 9.2 and Vulkan already installed)

#### Build a Docker image (option 1):
Install Docker and its dependencies:
```bash
sudo ./docker/ubuntu/xenial/vulkan-base/pre_docker_install.sh
```
Build docker image (Requires about 15 minutes):
```bash
sudo ./docker/ubuntu/xenial/vulkan-base/build.sh
```
Run container over the image:
```bash
sudo ./docker/ubuntu/xenial/vulkan-base/run.sh
```
Change to head2head directory (inside the container):
```bash
cd head2head
```

#### Create a Conda environment (option 2):
Create a conda environment, using the provided ```conda-env.txt``` file.
```bash
conda create --name head2head --file conda-env.txt
```
Activate the environment.
```bash
conda activate head2head
```
Install facenet-pytorch, insightface and mxnet with pip (inside the environment):
```bash
pip install insightface mxnet-cu92mkl facenet-pytorch
```

## head2head Dataset

#### Video data visualisation after face detection and cropping

We have trained and tested head2head on the seven target identities shown below:

![](imgs/head2headDataset_identities.gif)

#### Download head2head Dataset

- Link to the seven original video files, before ROI extraction: [\[original_videos.zip\]](https://www.dropbox.com/s/moh71pvtll9n9ye/original_videos.zip?dl=1). The corresponding YouTube urls, along with the start and stop timestamps are listed in ```datasets/head2headDataset/urls.txt``` file.
- Link to full dataset, with the extracted ROI frames and 3D reconstruction data (NMFCs, landmarks, expression, identity and camera parameters): [\[dataset.zip\]](https://www.dropbox.com/s/saimhaftz27fjqt/dataset.zip?dl=1)

Alternatively, you can download head2head Dataset, running:

```bash
python scripts/download/download_dataset.py
```

It will be placed under ```datasets/head2headDataset```.

#### head2head Dataset structure

We split the original video of each identity into one training and one test sequence. We place about one third of the total number of frames in the test split. In this way, we are able to use these frames as ground truth, when testing the model in a self reenactment scenario.

```
head2headDataset ----- original_videos
                   |
                   --- dataset ----- train ----- exp_coeffs (expression vectors)
                                 |           |
                                 --- test    --- id_coeffs (identity vector)
                                    (same    |
                                  structure) --- images (ROI RGB frames)
                                             |
                                             --- landmarks (5 facial landmarks)
                                             |
                                             --- misc (camera parameters - pose)
                                             |
                                             --- nmfcs (GAN conditional input)
```

## Create your Dataset

You can create your own dataset from .mp4 video files. For that, first we do **face detection**, which returns a fixed bounding box that is used to extract the ROI, around the face. Then, we perform **3D face reconstruction** and compute the NMFC images, one for each frame of the video.

#### Face detection (tracking)

In order to perform face detection and crop the facial region from a single .mp4 file or a directory with multiple files, run:

```bash
python preprocessing/detect.py --original_videos_path <videos_path> --dataset_name <dataset_name> --split <split>
```

- ```<videos_path>``` is the path to the original .mp4 file, or a directory of .mp4 files. (default: ```datasets/head2headDataset/original_videos```)

- ```<dataset_name>``` is the name to be given to the dataset. (default: ```head2headDataset```)

- ```<split>``` is the data split to place the file(s). If set to ```train```, the videos-identities can be used as target, but the last one third of the frames is placed in the test set, enabling self reenactment experiments. When set to ```test```, the videos-identities can be used only as source and no frames are placed in the training set. (default: ```train```)

#### Face reconstruction

Make sure you have downloaded the required models and files for 3D facial fitting, with:

```bash
python scripts/download_preprocessing_files.py
```

To perform 3D facial reconstruction and compute the NMFC images of all videos-identities in the dataset, please run:

```bash
python preprocessing/reconstruct.py --dataset_name <dataset_name>
```

Please execute the command above each time you use the face detection script to add new identities in the ```<dataset_name>``` dataset.

## Train a head2head model

First, compile FlowNet2:
```bash
python scripts/compile_flownet2.py
```

In order to train a new person-specific model from scratch, use:

```bash
./scripts/train/train_on_target.sh <target_name> <dataset_name>
```

where ```<target_name>``` is the name of the target identity, which should have been already placed in the ```<dataset_name>``` dataset, after processing the original video file: ```<target_name>.mp4```.

## Test self reenactment

In self reenactment, the target person is also used as source. In this way we have access to the ground truth video, which provides a means to evaluate the performance of our model.

The following commands generates a video, using as driving input (source video) the kept out, test frames of ```<target_name>```:

```bash
./scripts/test/test_self_reenactment_on_target.sh <target_name> <dataset_name>
```

Synthesised videos are saved under the ```./results``` directory.

## Test head-to-head reenactment

For transferring the expressions and head pose from a source person, to a target person in our dataset, first we compute the NMFC frames that correspond to the source video, using the 3DMM identity coefficients computed from the target. For better quality, we adapt the mean Scale and Translation camera parameters of the source to the target. Then, we generate the synthetic video, using these NMFC frames as conditional input.

Given a ```<source_name>``` and a ```<target_name>``` from dataset ```<dataset_name>```, head to head reenactment results are generated after running:
```bash
./scripts/test/test_reenactment_from_source_to_target.sh <source_name> <target_name> <dataset_name>
```

## Real-time head-to-head reenactment
