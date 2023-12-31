# Multianimal DLC: Tracking dyadic free social interaction between C57BL/6 mice.

This network was trained by manually selecting 300 frames obtained from 4 videos recorded in 2 different sociability chambers.
In 70% of the frames, the dyad was interacting closely, while in the remaining 30%, both animals were far from each other.
My goal was to train the network emphasizing social interaction over global tracking.

In each frame, body parts were labeled as follows: hocico (snout), cabeza (head), cuerposuperior (upper body), cuerpoinferior (lower body), colabase (base of the tail), colapunta (tip of the tail). In the config file, identity was set to "false"  to allow the network to deduce the animal's identities on its own.
The network was trained for 200.000 iterations using Google Colab.

FILES:
- config.yaml: the configuration file used for training the network. 
- CollectedData_Juan.csv#L304: information regarding the labelled data.
- dyadic_dlcrnetms5_KOdyadicJun13shuffle1_200000_el_filtered_id_labeled%20(2).mp4: resulting video (can be viewed raw).
- log.txt: the training log in Google Colab.
- CombinedEvaluation-results.xlsx: the evaluation results.
- glossary: information about some difficult technical terms I encountered in the process. 
