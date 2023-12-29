# Dean Kelley TCSS 702 Master's Capstone Project
The purpose of this project was to utilise the skills and tools acquired throughout my time as a graduate student at the University of Washington to produce a data processing pipeline and a model to accurately classify a status for cancer hormone receptors for a given tissue sample whole slide image (WSI).

To run the model on the TCGA dataset clone this repository to your working directory, start a Jupyter Hub session in the same directory, and follow the instructions listed below:
## Data Process Pipeline
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

The processor utilizes parallel processing
  
## Prediction Model
