# DLC
Tracking dyadic social interaction between mice with multi-animal DLC

13/6/23:

- instalo DLC, creo el proyecto, cropeo 4 videos con la GUI y extraigo frames automáticamente con el algoritmo kmeans (2 chamber A, 2 chamber B)

14/6/23:

- del total de frames selecciono manualmente 300, de los cuales 200 tienen a la díada cerca o interactuando (nombres de archivos inician con C_ de close) y 100 no. De cada video queda así: OBS1: 38C, 25 otros; OBS2: 67C, 25 otros; OBS3: 34C, 25 otros; OBS4: 61C, 25 otros.
La idea es entrenar la red haciendo hincapié en las conductas target de interacción social y no tanto en el trackeo global del animal,
que fue lo que se hizo en anteriores oportunidades y no salió muy bien (ej: confundía la ID de los animales).

- hago el labeling de las partes del cuerpo en cada frame. Defino las bodyparts como: hocico, cabeza, cuerposuperior, cuerpoinferior, colabase y colapunta. Seteo a identity como "false" en el archivo config, ya que no voy a mantener la identidad de los animales entre frames y eso lo va a deducir la red entrenada.

En Napari se debe seleccionar la opción "save selected layers" y no la "save all layers" (cuelga el programa y no lo guarda).
Se crean 2 archivos: CollectedData_Juan.csv y CollectedData_Juan.h5 (este último será usado en el entrenamiento de la red).
Repito el labeling para los 4 videos, obteniendo 4 archivos .csv y 4 .h5. 

Consulta: al momento de crear el training set, toma la info de esos 4 .h5? Respuesta: SÍ (ver 16/6)

16/6/23:

- chequeo las labels ingresando: deeplabcut.check_labels(config_path, visualizeindividuals=True) en el CMD. Eso crea 4 subcarpetas con terminación _labeled en la carpeta labeled-data. Parecen estar ok y se crearon para los 4 videos labeleados.

- para crear el training set cargo la carpeta del proyecto a Google drive en mi cuenta ueharajm.dlc@gmail.com. Creo un Collab llamado "KOdyadic-Juan-2023-06-13". Descargo el config.yaml y le cambio el project_path a /content/drive/MyDrive/KOdyadic-Juan-2023-06-13. Lo resubo al drive.

- Ya en el Collab, cambio la ruta de acceso a path_config_file = '/content/drive/MyDrive/KOdyadic-Juan-2023-06-13/config.yaml'.
Más abajo, le doy a deeplabcut.create_multianimaltraining_dataset(path_config_file, Shuffles=[shuffle], net_type="dlcrnet_ms5",windows2linux=True).

windows2linux=True se pone porque los frames ya se labelearon previamente.

Eso crea el training dataset con los siguientes parámetros:  Shuffle: 1 TrainFraction:  0.95 (285 en total). Editar la variable shuffle en caso de querer re-entrenar la red. Se crean las siguientes carpetas:

1) en dlc-models: iteration-0. Esta carpeta es importante porque en la subcarpeta TRAIN y en el archivo pose_cfg.yaml se puede cambiar el parámetro init_weights (inicialmente seteado a resnet) para retomar el entrenamiento. Se debe descargar el archivo pose_cfg.yaml y modificar el parámetro "init_weights" para que apunte a la ruta del snapshot de la última iteración en vez de a la red preentrenada.
Ej: init_weights: /content/drive/My Drive/your project/dlc-models/iteration-n/NOMBRE/train/snapshot-30000. ELIMINAR LA EXTENSIÓN QUE SIGE AL NÚMERO DE SNAPSHOT.
Para que tome ese snapshot, el archivo debe ser reemplazado en Colab DESPUES de haber creado el training dataset porque si se sube antes y luego se crea el training dataset, el archivo pose es reemplazado por resnet nuevamente.

2) en training-datasets: iteration-0: UnaugmentedDataSet_KOdyadicJun13. En el archivo CollectedData_Juan.csv se comprueba que el data set fue creado con los labels de los 4 archivos. También se crearon los archivos .h5 y .pickle.

- En Collab voy a START TRAINING y cambio:
deeplabcut.train_network(path_config_file, shuffle=shuffle, displayiters=100,saveiters=1000, maxiters=75000, allow_growth=True).
por: deeplabcut.train_network(path_config_file, shuffle=shuffle, displayiters=1000,saveiters=5000, maxiters=100000, allow_growth=True). No excederse de 100000 en maxiters porque dicen que "overfitea" y afecta la performance. Pongo saveiters=5000 por si el Collab se desconecta, cuestión de no perder 10000 iteraciones. Empiezo a entrenar.

20/6/23:

- Reemplazo en el pose_cfg.yaml el parámetro init_weights por SNAPSHOT-85000  sin extensión.
- Reemplazo el archivo en el drive y le doy a entrenar nuevamente.
- Retoma desde el snapshot indicado pero continúa más allá de las 100000 iteraciones indicadas en el Colab como maxiters. Ésto sucede porque también hay que cambiar el valor en el archivo pose_cfg.yaml, en la última línea del parámetro multistep:

multi_step:
- - 0.0001
  - 7500
- - 5.0e-05
  - 12000
- - 1.0e-05
  - 200000  <<<<<<<<<<<<<<  Lo cambio a 150000.

- No se soluciona el problema de maxiters, sigue iterando. Dejo corriendo hasta 200000, supuestamente para automáticamente.
- El problema persiste. Me pongo en contacto por el foro de DLC.

21/6/23:

- Pese a que detuve el entrenamiento manualmente, voy a START EVALUATING para chequear si me deja proceder. Me deja seguir.
- Se crea la carpeta evaluation-results para la iteración 0. Para analizar estos datos, me remito al manual de multianimal:

The evaluation results for each shuffle of the training dataset are stored in a unique subdirectory in a newly created directory ‘evaluation-results’ in the project directory.
The user can visually inspect if the distance between the labeled and the predicted body parts are acceptable.


Los resultados de la evaluación son los siguientes:

Training iterations: %Training dataset Shuffle number Train error(px) Test error(px) p-cutoff used Train error with p-cutoff Test error with p-cutoff
200000 95 1 02.07 2.28 0.6 02.07 2.28

Average Euclidean distance to GT (ground truth, or the manually-placed label) per individual (in pixels; test-only)
individuals
raton1    2.435281
raton2    2.174187

Average Euclidean distance to GT per bodypart (in pixels; test-only)
bodyparts
cabeza            2.783565
colabase          1.996882
colapunta         2.030366
cuerpoinferior    2.677194
cuerposuperior    2.139069
hocico            2.492535

*** the average error per bodypart is the average of all the data (both test and train) with p-cutoff applied (i.e. not considering low-confidence predictions). ***

CONCLUSION 1/3: el train y test error son similares y parecen bajos.

La evaluación basada en imágenes la hago con la carpeta LabeledImages_DLC_dlcrnetms5_KOdyadicJun13shuffle1_200000_snapshot-200000.
Acá lo más importante es ver las imágenes que empiezan con Test (corresponde al 5% del training set donde la red intentó estimar la posición)
Ésto es importante para leer la imagen:
Manually set labels are displayed as a plus symbol “+” and the model’s predictions either as a dot (for predictions with a high likelyhood) or as an “x” (for predictions with a low likelyhood)

CONCLUSION 2/3: Las imágenes parecen estar bien, aunque del total del test-set sólo seleccionó 4 imágenes en las que los animales estaban cerca, pese a que representaban la mayoría del training-set.

LOCREF: en algunas imágenes del test la nube celeste se ve un poco corrida (ej: img02018.png_locref_test_1_0.95_snapshot-200000), lo que también pasa en algunas del train (ej: C_img01481.png_locref_train_1_0.95_snapshot-200000).
En líneas generales están bien.
PAF: las imágenes del test parecen estar bastante bien, discriminando la ID de los 2 animales, lo que también sucede en las del train.
SCMAP: son las mismas imágenes de LOCREF sin las flechas rojas.

CONCLUSION 3/3: el entrenamiento de la red parece estar bastante bien.

PUNTO DE DECISIÓN:

In the event of benchmarking with different shuffles of same training dataset, the user can provide multiple shuffle indices to evaluate the corresponding network.
If the generalization is not sufficient, the user might want to:

• check if the labels were imported correctly; i.e., invisible points are not labeled and the points of interest are labeled accurately
• make sure that the loss has already converged
• consider labeling additional images and make another iteration of the training data set

Para definir ésto voy a continuar el workflow (ANALIZAR y CREAR VIDEO LABELEADO) en un Colab paralelo (en la cuenta ueharajm_dlc@gmail.com) usando una carpeta sham, cuestión de que no me afecte los resultados hasta ahora.
Si el resultado no es bueno, lo elimino y retomo desde el checkpoint previo en ueharajm_dlc2@gmail.com, labeleando más frames o haciendo un nuevo shuffle para el training-set.

- Creo un checkpoint de la carpeta del Colab y la bajo a mi desktop. Lo nombro checkpoint_21062023 (incluye hasta iteración 200000 pero no finalizó el training).

- Redefino la variable videofile_path, que al comienzo del Colab apuntaba a TODA la carpeta de videos. Ahora apunta a un video específico. ANALYZE queda así:

#Redefino a videofile_path:
videofile_path = '/content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4'

deeplabcut.analyze_videos(path_config_file,videofile_path, shuffle=shuffle, videotype='.mp4')

- Luego le doy play a ANALYZE. El resultado es el siguiente:

Start Analyzing my video(s)!
Using snapshot-200000 for model /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/dlc-models/iteration-0/KOdyadicJun13-trainset95shuffle1
/usr/local/lib/python3.10/dist-packages/tensorflow/python/keras/engine/base_layer_v1.py:1694: UserWarning: `layer.apply` is deprecated and will be removed in a future version. Please use `layer.__call__` method instead.
  warnings.warn('`layer.apply` is deprecated and '
Activating extracting of PAFs
Starting to analyze %  /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4
Loading  /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4
Duration of video [s]:  599.95 , recorded with  19.65 fps!
Overall # of frames:  11789  found with (before cropping) frame dimensions:  320 240
Starting to extract posture from the video(s) with batchsize: 8
100%|██████████| 11789/11789 [01:48<00:00, 108.28it/s]
Video Analyzed. Saving results in /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos...
Using snapshot-200000 for model /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/dlc-models/iteration-0/KOdyadicJun13-trainset95shuffle1
Processing...  /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4
Analyzing /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZDLC_dlcrnetms5_KOdyadicJun13shuffle1_200000.h5
100%|██████████| 11789/11789 [00:35<00:00, 336.11it/s]
11789it [00:22, 531.29it/s]
The tracklets were created (i.e., under the hood deeplabcut.convert_detections2tracklets was run). Now you can 'refine_tracklets' in the GUI, or run 'deeplabcut.stitch_tracklets'.
Processing...  /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4
/usr/local/lib/python3.10/dist-packages/deeplabcut/refine_training_dataset/stitch.py:138: FutureWarning: Unlike other reduction functions (e.g. `skew`, `kurtosis`), the default behavior of `mode` typically preserves the axis it acts along. In SciPy 1.11.0, this behavior will change: the default value of `keepdims` will become False, the `axis` over which the statistic is taken will be eliminated, and the value None will no longer be accepted. Set `keepdims` to True or False to avoid this warning.
  return mode(self.data[..., 3], axis=None, nan_policy="omit")[0][0]
100%|██████████| 60/60 [00:00<00:00, 2299.36it/s]
The videos are analyzed. Time to assemble animals and track 'em...
 Call 'create_video_with_all_detections' to check multi-animal detection quality before tracking.
If the tracking is not satisfactory for some videos, consider expanding the training set. You can use the function 'extract_outlier_frames' to extract a few representative outlier frames.
DLC_dlcrnetms5_KOdyadicJun13shuffle1_200000

- Le doy play a CREATE VIDEO WITH ALL DETECTIONS.
##### PROTIP: #####
## look at the output video; if the pose estimation (i.e. key points)
## don't look good, don't proceed with tracking - add more data to your training set and re-train!

#EDIT: let's check a specific video (PLEASE EDIT VIDEO PATH):
Specific_videofile = '/content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4'

#don't edit:
deeplabcut.create_video_with_all_detections(path_config_file, [Specific_videofile], shuffle=shuffle)

deeplabcut.utils.plot_edge_affinity_distributions

El resultado es el siguiente:

Creating labeled video for  OBS1_DEMO1_LIBRE_SZ
100%|██████████| 11789/11789 [00:43<00:00, 270.57it/s]
<function deeplabcut.utils.plotting.plot_edge_affinity_distributions(eval_pickle_file, include_bodyparts='all', output_name='', figsize=(10, 7))>

EN RESUMEN: del video OBS1 analizó 11789 frames de 320x240. En la carpeta videos crea los archivos .pickle y .h5.
Creó un archivo de video nuevo llamado OBS1_DEMO1_LIBRE_SZDLC_dlcrnetms5_KOdyadicJun13shuffle1_200000_full. El video parece trackear bien las partes de cada animal.
PROBLEMA: reflejos en el chamber. Deben extraerse frames y corregirse.

- Agrego el apartado EVALUATE AFFINITY DISTRIBUTIONS (ver FIG S4: https://www.biorxiv.org/content/10.1101/2021.04.30.442096v1.full.pdf). FALLA.

- Agrego el apartado EXTRACT OUTLIER FRAMES:

pickle_file = '/content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZDLC_dlcrnetms5_KOdyadicJun13shuffle1_200000_full.pickle'
deeplabcut.find_outliers_in_raw_data(path_config_file, pickle_file, Specific_videofile)

El resultado es el siguiente:

Frames from video OBS1_DEMO1_LIBRE_SZ  already extracted (more will be added)!
Loading video...
Cropping coords: None
Duration of video [s]:  599.9491094147584 , recorded @  19.65 fps!
Overall # of frames:  11789 with (cropped) frame dimensions:
Kmeans-quantization based extracting of frames from 0.0  seconds to 599.95  seconds.
Extracting and downsampling... 231  frames from the video.
231it [00:18, 12.33it/s]
/usr/local/lib/python3.10/dist-packages/sklearn/cluster/_kmeans.py:870: FutureWarning: The default value of `n_init` will change from 3 to 'auto' in 1.4. Set the value of `n_init` explicitly to suppress the warning
  warnings.warn(
Kmeans clustering ... (this might take a while)
Let's select frames indices: [2994, 10834, 3807, 10420, 7123, 858, 6852, 1417, 4995, 8247, 3898, 4205, 10402, 11460, 6788, 7683, 181, 11012, 10339, 10172, 7297, 185, 7508, 1539, 7996, 49, 6468, 2587, 6513, 10467, 7589, 5579, 9240, 22, 9965, 232, 939, 10900, 1114, 3074, 198, 2649, 7240, 10470, 6920, 523, 8877, 4842, 193, 3792, 6771, 3128, 4832, 5482, 10904, 1686, 3740, 10725, 6257, 5617, 2342, 3994, 11076, 1146, 10298, 4421, 327, 8885, 632, 5616, 7568, 186, 5613, 5889, 387, 3798, 4214, 9877, 3884, 3828, 10409, 8305, 2598, 11088, 7611, 3663, 1533, 8897, 4393, 2653, 8920, 190, 6340, 9760, 541, 166, 920, 1544, 9, 9003, 6905, 2589, 2610, 6929, 2597, 5026, 197, 4210, 4206, 2616, 6460, 1719, 6345, 2628, 8362, 2139, 3445, 1113, 6462, 3501, 10333, 2361, 938, 5425, 3975, 10784, 10509, 3161, 1415, 3794, 10472, 5444, 3850, 7309, 7604, 2644, 998, 3053, 3788, 7550, 160, 172, 3829, 51, 2611, 10805, 940, 10399, 7125, 6309, 2580, 3821, 9835, 2583, 6466, 1918, 2622, 3900, 5174, 10393, 5609, 10404, 3131, 178, 10414, 2617, 9584, 7317]
Attempting to create a symbolic link of the video ...
Video /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4 already exists. Skipping...
New videos were added to the project! Use the function 'extract_frames' to select frames for labeling.
The outlier frames are extracted. They are stored in the subdirectory labeled-data\OBS1_DEMO1_LIBRE_SZ.
Once you extracted frames for all videos, use 'refine_labels' to manually correct the labels.

EN RESUMEN: extrajo 231 frames que son outliers del video OBS1 y los incluyó en la carpeta labeled-data\OBS1_DEMO1_LIBRE_SZ.
HACER LO MISMO PARA TODOS LOS VIDEOS ANTES DE ENTRENAR UN NUEVO DATA-SET!!!!!

- Corro ASSEMBLE ANIMALS:

#Check and edit:
numAnimals = 2 #how many animals do you expect to find?
tracktype= 'ellipse' #box, skeleton, ellipse:
#-- ellipse is recommended, unless you have a single-point ma project, then use BOX!

#Optional:
#imagine you tracked a point that is not useful for assembly,
#like a tail tip that is far from the body, consider dropping it for this step (it's still used later)!
#To drop it, uncomment the next line TWO lines and add your parts(s):

bodypart= 'colapunta'
deeplabcut.convert_detections2tracklets(path_config_file, videofile_path, videotype=VideoType, shuffle=shuffle, overwrite=True, ignore_bodyparts=[bodypart])

#OR don't drop, just click RUN:
#deeplabcut.convert_detections2tracklets(path_config_file, videofile_path, videotype=VideoType,
#                                        shuffle=shuffle, overwrite=True)

deeplabcut.stitch_tracklets(path_config_file, videofile_path, shuffle=shuffle, track_method=tracktype, n_tracks=numAnimals)

El resultado es el siguiente:

Using snapshot-200000 for model /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/dlc-models/iteration-0/KOdyadicJun13-trainset95shuffle1
Processing...  /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4
Analyzing /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZDLC_dlcrnetms5_KOdyadicJun13shuffle1_200000.h5
100%|██████████| 11789/11789 [00:29<00:00, 397.57it/s]
11789it [00:17, 683.57it/s]
The tracklets were created (i.e., under the hood deeplabcut.convert_detections2tracklets was run). Now you can 'refine_tracklets' in the GUI, or run 'deeplabcut.stitch_tracklets'.
Processing...  /content/drive/MyDrive/KOdyadic-Juan-2023-06-13/videos/OBS1_DEMO1_LIBRE_SZ.mp4
/usr/local/lib/python3.10/dist-packages/deeplabcut/refine_training_dataset/stitch.py:138: FutureWarning: Unlike other reduction functions (e.g. `skew`, `kurtosis`), the default behavior of `mode` typically preserves the axis it acts along. In SciPy 1.11.0, this behavior will change: the default value of `keepdims` will become False, the `axis` over which the statistic is taken will be eliminated, and the value None will no longer be accepted. Set `keepdims` to True or False to avoid this warning.
  return mode(self.data[..., 3], axis=None, nan_policy="omit")[0][0]
100%|██████████| 87/87 [00:00<00:00, 1793.24it/s]
/usr/local/lib/python3.10/dist-packages/deeplabcut/refine_training_dataset/stitch.py:690: UserWarning: No optimal solution found. Employing black magic...
  warnings.warn("No optimal solution found. Employing black magic...")

CONCLUSION FINAL: el video resultante es mejor que los obtenidos en anteriores veces pero hay frames en donde la identidad de los animales se sigue confundiendo. Se podría reentrenar la red con los frames outliers arrojados y labelearlos nuevamente. El manual recomienda lo siguiente:

If the identity of two mice is frequently confused and not accurately maintained throughout your DeepLabCut-trained video, there could be several factors contributing to this issue. Here are some possible reasons and suggestions to address them:

1. Training Data Quality: Insufficient or poor-quality training data can lead to inaccurate results. Ensure that you have a diverse and representative dataset with enough examples of both mice. Ideally, the training data should cover various angles, lighting conditions, and positions to capture the full range of appearances for each mouse.

2. Annotation Errors: Double-check your annotation process to ensure accurate labeling of the mice. Mistakes in annotation, such as mislabeling or inaccurate marking of body parts, can significantly impact the tracking performance. Review your annotated frames and correct any potential errors.

3. Model Configuration: The performance of DeepLabCut can be affected by the chosen model architecture and its configuration. Ensure you have selected an appropriate model for your specific tracking task and adjust the model parameters accordingly. Experiment with different model architectures, such as increasing the number of layers or changing the backbone network, to improve performance.

4. Insufficient Training: Deep learning models require sufficient training to learn the intricacies of the task. Make sure you have trained your model for an adequate number of iterations. Consider increasing the number of training iterations and monitoring the training progress to see if the performance improves over time.

5. Tracking Algorithm Selection: DeepLabCut provides options for different tracking algorithms. Experiment with different algorithms, such as "centroid" or "skeleton," to find the one that best suits your specific tracking scenario. Some algorithms may perform better for distinguishing between the two mice.

6. Tracking Parameter Adjustment: Adjusting the tracking parameters can sometimes improve the accuracy of identity maintenance. Parameters such as the maximum distance between body parts or the minimum likelihood threshold for body part detection can be adjusted to fine-tune the tracking performance.

7. Post-processing Techniques: Consider applying post-processing techniques to refine the tracking results. These can include smoothing filters, temporal consistency checks, or using additional information (e.g., prior knowledge about mouse behavior) to assist in disambiguating the identities of the mice.

Remember, tracking accuracy can be a challenging task, and achieving perfect results may not always be feasible. It is important to iterate, experiment, and refine your approach based on the specific challenges and requirements of your tracking scenario.
