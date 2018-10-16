# Surrogate Supervision for Nodule Detection FPR


## Data

### 1. FPR:
The task of FPR for nodule detection is to label each nodule candidate (3D patch) as either nodule or non-nodule.
We first split the LIDC dataset[1] into the training, validation, and test subsets with 733, 95, and 190 CT series, respectively. To generate nodule candidates, we trained a 3D faster RCNN model using the training subset. We then ran this model against all 1,018 CT series from LIDC. The resulting candidate locations were combined with the ground truth locations provided by LIDC. Finally, we grouped these TP, GT, and FP locations according to series UIDs for training, validation, and testing purposes.

For preprocessing, 3D patches are clipped from -1200 to 600 in Hounsfield unit.
Their values are then normalized into the range of -1 to 1.
All 3D patches are resampled to the spacing of 1mm x 1mm x 1mm.
Finally they are center cropped to the shape of 32 x 32 x 32 along z, y, and x axis.

To alleviate the class-imbalance problem, the ratio between non-nodule patches and nodule patches are kept to 8:1 by random down sampling of non-nodule patches. We further re-sample the nodule patches as follows:
(1) if the diameter of the nodule is smaller than 3mm, we remove it;
(2) if the nodule diameter is between 3mm and 25mm, we upsample it by 2 times;
(3) if the nodule diameter is between 25mm and 50mm, we directly feed it into the model without upsampling.

### 2. Surrogate Supervision GAN:
For surrogate supervision task, the GAN model is trained using both the 784,241 FPR training patches (*Tr<sup>l</sup>*) and the 4,750 FPR validation patches (*V<sup>l</sup>*). It is then tested on the 3,059 FPR testing patches (*Te<sup>l</sup>*).
The preprocessing and upsampling follows the same methods adopted by FPR model.


## Architectures

### 1. Surrogate Supervision GAN:
![GAN Architecture](./FPR_GAN3.png)

This surrogate supervision GAN model follows the same training and inference procedures as proposed in the original GAN paper [3]. Once trained, alpha1, alpha2, alpha3, alpha4 will all become 1 and the discriminator network will reduce to the FPR network as shown in the figure below.

### 2. FPR:
![FPR Architecture](./FPR2.png)


## Training Details
Both FPR and surrogate supervision GAN were implemented using Tensorflow [4].
They were both trained on an NVIDIA 1080Ti GPU.

### 1. Surrogate Supervision GAN:
Batch size is 64. Total number of training iterations is 96,000.
Wasserstein loss [6] was used because of its stability.
The dimension of the latent variables is 100.
An Adam optimizer was used for configuring the learning rate with beta1 of 0.5 and beta2 of 0.99.
The learning rate started at 2e-4. 

Most importantly, the training was conducted in a progressive fashion as suggested by [5] to generate higher quality 3D patches.
Notice there are 4 alphas in both the discriminator and generator network as shown in the figure above.
These alphas were used to configure the training phase of the entire GAN model by controlling the input 3D patch and output fake 3D patch resolution.
Specifically, when alpha1 is turned on, the resolution is 4x4x4;
when alpha2 is turned on, the resolution is 8x8x8;
when alpha3 is turned on, the resolution is 16x16x16;
when alpha4 is turned on, the resolution is 32x32x32;
The slope for gradually turning on these alphas is 0.002.
After 5,000 training steps, alpha1 was turned on;
after 12,000 training steps, alpha2 was turned on;
after 19,000 training steps, alpha3 was turned on;
and after 26,000 training steps, alpha4 was turned on.

### 2. FPR:
Batch size is 128. Total number of training iterations is 196,000.
An SGD optimizer was used for configuring the learning rate.
The learning rate started at 3e-4, then decreased by 10 times at training step of 120,000.

The model was trained from scratch, pretrained from the discriminator of the surrogate supervision GAN model using 10%, 25%, 50%, and 100% of all the 784,241 3D training patches (*Tr<sup>l</sup>*). There has been altogether 8 FPR models been trained.


## Results
![FPR result](./FPR_result3.png)

The performances of FPR models trained from scratch and pretrained from the discriminator of the surrogate supervision GAN model are shown above. These performances are measured in terms of FROC AuC (up to 3 false positives).
The x axis is the percentage of training data used for FPR model.


## References

[1]: Armato III, Samuel G., et al. "The lung image database consortium (LIDC) and image database resource initiative (IDRI): a completed reference database of lung nodules on CT scans." Medical physics 38.2 (2011): 915-931.

[2]: Arnaud Arindra Adiyoso Setio, et al. "Validation, comparison, and combination of algorithms for automatic detection of pulmonary nodules in computed tomography images: the LUNA16 challenge" CoRR, vol. abs/1612.08012, 2016.

[3]: I. J. Goodfellow, et al. "Generative adversarial nets" In Proceedings of NIPS, pages 2672–2680, 2014.

[4]: Abadi, Martín, et al. "Tensorflow: a system for large-scale machine learning." OSDI. Vol. 16. 2016.

[5]: Tero Karras, et al. “Progressive growing of gans for improved quality, stability, and variation,” CoRR, vol. abs/1710.10196, 2017.

[6]: Martin Arjovsky, et al. “Wasserstein gan,” arXiv preprint arXiv:1701.07875, 2017.



