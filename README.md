# Dean Kelley TCSS 702 Master's Capstone Project
The purpose of this project was to utilize the skills and tools acquired throughout my time as a graduate student at the University of Washington to produce a data processing pipeline and a model to accurately classify a status for cancer hormone receptors for a given tissue sample whole slide image (WSI).

## Installation
Prior to running the data processing pipeline and model, it is necessary to install the essential packages. These include:
1. Openslide
2. numpy
3. pandas
4. shutil
5. PIL
6. matplotlib
7. sklearn
8. torch
9. torchvision
10. cv2
11. Jupyter
12. tqdm


To run the model on the TCGA dataset, clone this repository to your working directory, start a Jupyter Lab or Notebook session in that same directory, and follow the instructions listed below:

**Special note**: two separate machines were used to process the data and run the model. Some or many variables may be reused between the programs and will need to be fixed if everything will be done on a single machine.

## Data Processing Pipeline
Open Data_Preprocess.ipynb inside Jupyter Hub and Run through the cells. Under the cell Split Manifest, it is possible to download the dataset in segments should one not want to process everything simultaneously. This can be seen below Notice the line 'list_df' uses split and is currently set to use the whole dataset. Changing the value of 1 to another number, n, will split it into n parts. Subsequently, df_manifest_slice will need to be updated as different subsections are desired.

```
manifest_filename = 'slides/gdc_manifest_Tissue_Slides_non_annotated.txt'
manifest_filename_diag = 'slides/gdc_manifest_Tissue_Slides_diagnostic.txt'
df_manifest = pd.read_csv(manifest_filename, sep='\t')
df_manifest_diag = pd.read_csv(manifest_filename_diag, sep='\t')
df_manifest_all = pd.concat([df_manifest, df_manifest_diag]).drop_duplicates().reset_index(drop=True)
print(f'Number of files: {len(df_manifest_all)}')

list_df = np.array_split(df_manifest_all, 1)

df_manifest_slice = list_df[0]

df_manifest_slice.to_csv('slides/gdc_manifest_slice.txt', index=False, sep='\t')
print(f'Number of files in slice: {len(df_manifest_slice)}')
df_manifest_slice.head()
```

Running the Download cell will then download each of the items in df_manifest_slice.

Once the files are downloaded, a data frame is produced that can be used later by the data processor and model.

To begin splitting the WSIs into tiles, a new data frame called images_to_process is created which will keep track of the WSIs that still need to be processed in the event of a stop to the process so that it can be resumed.

The processor utilizes parallel processing to reduce processing time. The number of processors being utilized is determined by taking the floor of the square root of available CPU processors as shown by:
```
process_count = math.isqrt(len(os.sched_getaffinity(0)))
```
Manually adjust this number to fit your system's needs.

Once the WSIs are broken up into their respective tiles, they will be saved if they are 256x256 pixels, are not blurry, and the standard deviation of the channel means is greater than 5. The last requirement filters out grey shadows that provide no information. 
After the usable tiles are saved, each tile's file location is saved to a CSV file for a respective WSI, and its file location is written to the overall model data frame.

Finally, to reduce space, the original WSI image file is removed.

If the files are not on the same machine, transfer them now.

In the event that the images are processed in batches as described above, it is necessary to consolidate the new batch into the total population. After each batch is processed, run Update_New_Dataset.ipynb and run through each cell. This program integrates the new data into the dataset and then rejects entries that have an insufficient number of tiles.

## Inference Model
The Jupyter Notebook file Final_CrosVal_UnGated_SEResNeXt50_AMP_Model_for_TCGA_Capstone.ipynb contains the inference model, as well as the final data preparation steps. Once opened, the desired hormone receptor (HR) to run the model on is selected by updating the 'label' variable. 
```
label = 'ER_Label'
```
A tile consisting of the dataset's average pixel value is then produced utilizing multiprocessing. This may take a considerable amount of time, but once completed the resulting image is stored for later use. Next, the training and test datasets can be created using a stratified split based on the desired label. With the training and test datasets produced, the model is created by first establishing the MIL dataloader before running the model itself. To obtain the best performance of the dataloader, the optimal number of workers is determined by running a series of tests and utilizing the most efficient worker count. This process tests the dataloader utilizing an array of available CPU cores. The specific core counts to begin with can may be altered, but the default is set as two.
```
for num_workers in range(2, mp.cpu_count(), 2):
```
Running the optimal workers program can take a considerable amount of time as well, but once completed the results are stored for later use. With the model built, the following cells allow for running cross-validation, testing the model on the test dataset, and then running experiments to see the effects of ensemble/aggregation.

Since the model can take a significant amount of time to train depending on the GPU of the system, the model is saved after every epoch. To begin, there is a 'restart' variable that must be set to False. To restart at the most recent training epoch, one can change it to True.
```
restart = False # Restarting from a current epoch?
```

During the development of the model, an RTX2080Ti was used and models often required a whole day or more to complete, so the models were saved according to the date by a variable called 'date' found in the same initialization cell as the restart variable.
```
date = '2-12' # Checkpoint date
```
### Ensembles Test
The Ensemble test carries out a baseline test on a bag size of x before carrying out y tests of n bag sizes such that y = x // n. It will save the inference probabilities from each test and then take the max or min value depending on the given parameter e_choice and compute AUROC. It will then run this test five times and take the average before comparing the result against the baseline test AUROC.
