# Diagnosing Myocardial Infarction from 12-lead ECG data
## What is Myocardial Infarction?
Myocardial Infarction (MI) is a condition commonly referred to as heart attack. It often occurs when arteries supplying the heart with blood become blocked, causing heart muscle to die. Depending on the time before treatment, this can result in debilitating heart conditions or death and is one of the leading causes of death in high income countries.
## What is an ECG?
An electrocardiogram (ECG), measures the electrical signals in the heart and gives readings as time-series graphs of potential difference (voltage) across various directions through the heart.

<img src="https://github.com/CalumF/MI-ECG-12lead/blob/master/ECG_leads.png" alt="drawing" width="400"/>     <img src="https://github.com/CalumF/MI-ECG-12lead/blob/master/ECG_principle_slow.gif" alt="drawing" width="400"/>
## Why is automation important?
Early treatment of MI is key to increasing positive outcomes and drug treatments are available that can be administered by paramedics. In addition 12-lead ECGs are increasingly being added to, or are available in ambulances allowing diagnosis of MI by specially trained paramedics or potentially an automated system if required. This could increase access to early treatment and/or transportation directly to specialist facilities.
## Data
The PTB diagnostic ECG database was obtained from [physionet.org](https://physionet.org/content/ptbdb/1.0.0/), a repository of medical signal data. It contains 148 patients with MI and 52 healthy controls, each patient has between 1 and 5 high resolution 12-lead ECG recordings.

The patients were split during the initial steps into a train (70%) and test (30%) set to prevent the models from learning patient specific beat morphologies or signal abnormalities. This is an area that is covered only vaguely in papers that have attempted similar work on the same data set, often suggesting they may be splitting the data after segmenting the signals, allowing patient data leakage.

One of the issues with the data is a lack of information about the time from onset of symptoms, at which the recording was taken. As a result it is difficult to tell how suitable the data is for diagnosis in the early stages. Only including the first recording for each MI patient could help address this, additionally checking the date/time of further readings could allow use of more data if within a chosen time window.

<img src="https://github.com/CalumF/MI-ECG-12lead/blob/master/healthy-MI-ECGs.jpeg" alt="drawing" />

## Processing
The data was read using the wfdb library (https://pypi.org/project/wfdb/), the R peaks (an easily identifiable, prominent peak in the lead II signal), were detected using an algorithm from a library of ECG detectors (https://pypi.org/project/py-ecg-detectors/). The algorithm was run on lead I and II, only peaks identified in both leads were used and peak heights post-R were checked to try and eliminate anomalous R peak indexes.

Using the peak indexes, the data was sliced at set intervals either side to obtain individual beats. This method requires the patients have similar heart rates as otherwise, moving away from the R peak the heart beat features will become out of sync. This could potentially be solved by resampling proportional to heart rate. The beats were detrended using a linear least-squares fit. Since the magnitudes of signals can contain important diagnostic information I chose not to normalise the beats however this could have contributed to the model overfitting and normalisation should be tried.

<img src="https://github.com/CalumF/MI-ECG-12lead/blob/master/beat segmentation.png" alt="drawing" /> 

## Tensor Decomposition
Tensor decompositions of ECG signals have been reported previously and the idea of using an SVD-like method to reduce the leads into combinations that enable easy machine (or human) diagnosis was a key aspect that drew me to this project. In the context of modeling they prove unnecessary since they provide only a small reduction in training or evaluation times and there is significant reduction in accuracy.

In other contexts there are many possibilities. For instance, identifying the composite cartesian directions that have most changed in a person by differencing a previous healthy ECG and a new MI ECG. This could help further identify the axis along which conductivity has changed.

## Modelling
Modelling was performed using Keras with a TensorFlow backend. My model of choice was a long short-term memory (LSTM) neural network. These are recurrent neural networks that use gated units to store and recall information from data far back in time.

The training and test sets had 0.83 and 0.81 MI proportion and balanced class weighting was used to address this. Focal loss was also tried however it did not provide an improvement early on in modelling.

I experimented with various different network architectures and regularisation schemes, a common theme being immediate overfitting. In model training that showed test accuracy improving over the epochs (as one would hope), test loss generally increased. This is unusual and unexpected however there are potential explanations for it. It could be that as the optimisation improves training loss on “edge” cases, test loss is also reduced for these edge cases, increasing the proportion of correct predictions. For the easy training cases, loss could actually increase, and by more than the edge case reduction, however the prediction probabilities are far from the threshold so they remain correct. This leads to the increased test accuracy despite an increase in test loss.

<img src="https://github.com/CalumF/MI-ECG-12lead/blob/master/Model_Loss.png" alt="drawing" /> <img src="https://github.com/CalumF/MI-ECG-12lead/blob/master/Model_accuracy.png" alt="drawing" />

## Results
My final model was a Bidirectional LSTM with L2 regularisation, dropout and layer structure:

Layers | Nodes | dropout %
-------|-------|----------
LSTM 1| 64 | 0.5
LSTM 2| 32 | 0.5
LSTM 3| 8 | 0.5
Output| 1| 0

Loss: Binary Crossentropy, Optimiser: Nadam, epochs: 20, batch size: 64

Accuracy: 0.85 (0.5 baseline)

ROC AUC: 0.85

                  precision    recall  f1-score   support
 
               0       0.89      0.79      0.84      3340
               1       0.81      0.91      0.86      3340
 
        accuracy                           0.85      6680
       macro avg       0.85      0.85      0.85      6680
    weighted avg       0.85      0.85      0.85      6680

## Ideal/Future Work
There are two main areas of work to improve the project: one is the data cleaning/processing, the other is increasing the availability of information to the model.

#### Possible cleaning and processing steps
- Application of a band-pass filter
- Denoising using the db6 wavelet transform
- Detrending across multi-beat segments to reduce distortion and improve baseline correction
- Removing “follow up” recordings and focusing on ECGs from during the event
- Beat normalisation
- Resample beats according to heart rate to maintain feature synchronisation

#### Possible model changes
- Feed summary statistics directly such as heart rate, QRS duration etc
- Input multi-beat segments
- Retry focal loss
