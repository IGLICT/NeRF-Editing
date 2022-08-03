# NeRF-Editing: Geometry Editing of Neural Radiance Fields

![Teaser image](./img/teaser.gif)

## Abstract

Implicit neural rendering, especially Neural Radiance Field (NeRF), has shown great potential in novel view synthesis of a scene. However, current NeRF-based methods cannot enable users to perform user-controlled shape deformation in the scene. While existing works have proposed some approaches to modify the radiance field according to the user's constraints, the modification is limited to color editing or object translation and rotation. In this paper, we propose a method that allows users to perform controllable shape deformation on the implicit representation of the scene, and synthesizes the novel view images of the edited scene without re-training the network. Specifically, we establish a correspondence between the extracted explicit mesh representation and the implicit neural representation of the target scene. Users can first utilize well-developed mesh-based deformation methods to deform the mesh representation of the scene. Our method then utilizes user edits from the mesh representation to bend the camera rays by introducing a tetrahedra mesh as a proxy, obtaining the rendering results of the edited scene. Extensive experiments demonstrate that our framework can achieve ideal editing results not only on synthetic data, but also on real scenes captured by users.

## Environment
* Install [jittor](https://github.com/Jittor/jittor)
    <details>
    <summary> Other dependencies (click to expand) </summary>

    - opencv_python==4.5.2.52
    - imageio==2.17.0
    - trimesh==3.9.8 
    - numpy==1.19.2
    - pyhocon==0.3.57
    - icecream==2.1.0
    - tqdm==4.50.2
    - scipy==1.7.0
    - PyMCubes==0.1.2
    - natsort==8.1.0
    - tensorboardX-2.5

    </details>

We also use the pyrender to get the depth map.
```
pip install pyrender
```

* Download [OpenVolumeMesh](https://www.graphics.rwth-aachen.de/software/openvolumemesh/download/) to the `OpenVolumeMesh` folder
* Download [Eigen](https://eigen.tuxfamily.org/index.php?title=Main_Page) to the `volumeARAP_batch/Eigen` folder

## Data preparation
Suppose the image data is in the `$data_dir/images` folder, we first estimate the camera poses with [colmap](https://github.com/colmap/colmap). Then we process the camera poses with the command 
```
python process_colmap.py $data_dir $data_dir
```
Finally the data folder looks like
```
$data_dir
├── colmap_output.txt (colmap output)
├── database.db (colmap output)
├── images ($data_dir/images)
├── intrinsics.txt
├── pose
├── rgb
└── sparse (colmap output)
```


## Technological process

##### Training
we adopt the training strategy of [NeuS](https://github.com/Totoro97/NeuS).

```
python exp_runner.py --mode train --conf ./confs/wmask_mousy.conf --case mousy_neus
```

##### Extract mesh
 ```
python exp_runner.py --mode validate_mesh --conf <config_file> --case <case_name> --is_continue # use latest checkpoint
 ```

##### Render image before editing
```
python exp_runner.py --mode circle --conf ./confs/wmask_bear.conf --case bear_neus --is_continue  --obj_path ../../logs/bear_new_camera_neus/meshes/00300000.obj
```

##### Construct cage mesh
 ```
python exp_runner.py --mode validate_mesh --conf <config_file> --case <case_name> --is_continue --do_dilation
 ```

##### Construct tetrahedral mesh using [TetWild](https://github.com/Yixin-Hu/TetWild). 
```
./TetWild $input.obj
```
Note that we modify the tetrahedra storage format of Tetwild output. Therefore, please compile the `tetwild` in this repository following the instructions [here](https://github.com/Yixin-Hu/TetWild).

##### Change the output to `ovm` format.
```
./simple_mesh $input.txt $output.ovm
```
`simple_mesh` can be obtained using the `CMakeLists.txt` in the `OpenVolumeMesh` folder.

##### Editing
 Deform the extracted mesh *with any mesh editing tool*, and put the (sequence) mesh in `$deformed_dir` folder.

##### Propagate editing
Generate the controlpoint.txt to guide the deformation.
```
python barycentric_control_pts_jittor.py
```
Note that specify the `mesh_path` (extracted mesh), `tet_path` (tetrahedra mesh) and `deformed_dir`(deformed mesh sequence) first.

And the format of controlpoint.txt is listed below.

```
10 (Number of sequence)
N (Num of control points)
x1 y1 z1
x2 y2 z2
...
N (Num of control points)
x1 y1 z1
x2 y2 z2
...
.
.
.
N (Num of barycentric coordinate)
id1 id2 id3 id4 (vert index of this tet)
u1 v1 w1 z1
id1' id2' id3' id4'
u2 v2 w2 z2
...
```
Compile the `volumeARAP_batch` project to obtain `volumeARAP`, and deform the tetrehedra mesh.
```
./volumeARAP $ovm_path $control_pts.txt $output_dir 0
```
##### Rendering after editing
```
python exp_runner.py --mode circle --conf <config_file> --case <case_name> --is_continue --use_deform --reconstructed_mesh_file <tetwild_output_txt> --deformed_mesh_file <deformed_ovm> --obj_path <deformed_mesh_obj_path> (optional)
```

* fix camera (generate sequential editing results in a fixed camera)
```
python exp_runner.py --mode circle --conf <config_file> --case <case_name> --is_continue --use_deform --reconstructed_mesh_file <tetwild_output_txt> --deformed_mesh_file <deformed_ovm_dir> --obj_path <deformed_mesh_obj_dir> --fix_camera --savedir <save_dir>
```

## Acknowledgement
This code borrows heavily from [NeuS](https://github.com/Totoro97/NeuS).

## Citation

If you found this code useful please cite our work as:

```
@inproceedings{yuan2022nerf,
  title={NeRF-editing: geometry editing of neural radiance fields},
  author={Yuan, Yu-Jie and Sun, Yang-Tian and Lai, Yu-Kun and Ma, Yuewen and Jia, Rongfei and Gao, Lin},
  booktitle={Proceedings of the IEEE/CVF Conference on Computer Vision and Pattern Recognition},
  pages={18353--18364},
  year={2022}
}
```
