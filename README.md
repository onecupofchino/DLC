# Multianimal DLC: Tracking dyadic free social interaction between C57BL/6 mice.

This network was trained by manually selecting 300 frames obtained from 4 videos recorded at 2 different sociability chambers.
In 70% of the frames, the dyad was interacting closely. In the remaining 30%, both animals were far from each other.
My goal was to train the network emphasizing social interaction over global tracking.

In each frame, body parts were labelled as follows: hocico (snout), cabeza (head), cuerposuperior (upper body), cuerpoinferior (lower body), colabase (base of the tail), colapunta (tip of the tail). Identity was set to "false" in the config file to allow the network to deduce the animal's identities on its own.
The network was trained using Google Colab for 200.000 iterations.

The information regarding the labelled data can be accessed at CollectedData_Juan.csv#L304
The resulting video can be viewed raw at dyadic_dlcrnetms5_KOdyadicJun13shuffle1_200000_el_filtered_id_labeled%20(2).mp4
The training log in Google Colab can be views at log.txt
The evaluation results can be checked at CombinedEvaluation-results.xlsx
