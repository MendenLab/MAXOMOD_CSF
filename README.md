# MAXOMOD_CSF

In this repository, I divided the files on their respective modality. We have the following modalities:
- proteomics
- phosphoproteomics
- metabolomics
- smallRNA

IMPORTANT: I could not share patient data here, so you can not repeat the analyses of the projects without requesting the data first. Ana Galhoz, Michael Menden, and Laura Tzeplaeff have the data  that are needed to repeat the full analyses of the project. However, you can have a look at the original code that I wrote and recycle parts of it. Also, I created a 'mock' file for the proteomics part of the project, where I adjusted the code to use fake data so that you can still run all the code to learn how the pipeline works.

Let me quickly explain what is in each folder:

PROTEOMICS:

- real code differential abundancy analysis
This folder contains the code that I performed the analysis with. For privacy reasons, I could only include the code and not the data that I used. If you want to run the pipeline I created a mock version of the code that is basically the same analysis but using fake data.
- real code clustering
This folder contains the code that I performed the analysis with. For privacy reasons, I could only include the code and not the data that I used. If you want to run the pipeline I created a mock version of the code that is basically the same analysis but using fake data.
- mock version differential abundancy analysis
This mock version uses fake data that is used in the 'data' folder that you can find here. A knitted version of the Rmd file can be found in the MOCK_Proteomics_MAXOMOD.html file. More information on the analysis is in the respective folder in the README file. Note that if you run the Rmd code it will automatically create a 'plots' and 'results' folder where it puts the output data and figures.
- mock version clustering
This mock version uses fake data that is used in the 'data' folder that you can find here. A knitted version of the Rmd file can be found in the MOCK_Proteomics_MAXOMOD_clustering.html file. More information on the analysis is in the respective folder in the README file. Note that if you run the Rmd code it will automatically create a 'plots' and 'results' folder where it puts the output data and figures.

PHOSPHOPROTEOMICS

In this folder there is only an Rmd file and a knitted file. Both files can be used to look at the code that I wrote for the project. All the results are preliminary since I only worked on pre-processing of the data and performed differential expression analysis. The pathway analysis is not performed yet.

METABOLOMICS

In this folder there is only an Rmd file and a knitted file. Both files can be used to look at the code that I wrote for the project. All the results are preliminary since I only worked on pre-processing of the data and performed differential expression analysis. The pathway analysis is not performed yet.

SMALL RNA

The code in this folder was written by Ana Galhoz, not by me. For completeness sake and in agreement with Ana I put the code here anyway.

