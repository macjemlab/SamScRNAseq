[SRA Extraction.md](https://github.com/user-attachments/files/33984239/SRA.Extraction.md)
Batch downloading FASTQ files using the SRA toolkit 
================
-   [Workflow](#workflow)
-   [Finding raw sequencing data in GEO](#finding-raw-sequencing-data-in-geo)
-   [Downloading FASTQ files using the SRA toolkit](#downloading-fastq-files-using-the-sra-toolkit)
-   [Quality Control on FastQC](#quality-control-on-fastqc)
-   [Alignment and Quantification: Generating a Count Matrix](#alignment-and-quantification-generating-a-count-matrix)


Workflow
----------------------------------
1. Convert raw sequencing data to FastQ file using the SRA toolkit &rarr;  FastQ.  
2. FastQ becomes Fast QC &rarr; Perform Quality control (Read Trimming and Filtering if necessary)
3. Align and Quantify using CellRanger &rarr; CountMatrix

<img width="488" height="290" alt="Screenshot 2026-10-17 at 9 49 25 AM" src="https://github.com/user-attachments/assets/298e42f0-481a-4ed6-908e-3c500572a0d9" />


Finding raw sequencing data in GEO
----------------------------------
To eventually get the raw data in FASTQ format, we first need to first download the SRA Run files associated with each sample. If your sample has multiple runs then download all so that it can be further examined. A SRA file can be used by the NCBI's SRA toolkit as a set of "instructions" to construct the FASTQ file. First select the sample you wish to examine and under the Relations section select the SRA link. A new page will appear providing more in-depth data about the run(s), layout and selection. Under samples, select all runs, this will lead to the SRA run selector page.  
  
  **Check if samples are paired end or single end sequenced:**
  1. *Paired-end sequencing* : The sequencing instrument reads both ends of the same DNA or cDNA fragment, Produces Read 1 (R1) and Read 2 (R2), Both reads are associated with the same original fragment, Can provide additional information for alignment and transcript identification
  2. *Single-end sequencing* : The sequencing instrument reads only one end of each fragment, Typically produces one FASTQ file per run, There is no second read to pair with the first, Downstream software must be configured for single-end data when required

Downloading FASTQ files using the SRA toolkit
---------------------------------------------

In order to download the SRA files onto your machine, we use the NCBI's SRA toolkit, which lets us use the command line to download a specified SRA run. If you are using a Linux platform, you can type: `apt install sra-toolkit` in your command line to install the toolkit. You can read more about SRA toolkit here: <https://www.ncbi.nlm.nih.gov/books/NBK242621/> and at their github repo: <https://github.com/ncbi/sra-tools>. Link for the SRA toolkit <https://github.com/ncbi/sra-tools/wiki/01.-Downloading-SRA-Toolkit>


The toolkit works by first using the `prefetch` command to download the SRA file associated with the specified SRA run ID.

For example, sample GSM9221430 Lung immune ane epithelial cells from RA_1 patient has 8 SRA files (<https://www.ncbi.nlm.nih.gov/Traces/study/?acc=SAMN51091917&o=acc_s%3Aa>), to download the SRA files the command would be:

``` bash
mkdir C:\GEO_raw\GSE307297\sra
mkdir C:\GEO_raw\GSE307297\fastq
mkdir C:\GEO_raw\GSE307297\temp
```

These folders will keep the files organized:
- sra — downloaded SRA files.
- fastq — converted sequencing reads.
- temp — temporary files used during conversion.
Your SRA Toolkit can remain on your Desktop. The data do not need to be stored in the same folder as the toolkit.

To check that the toolkit on the Desktop works:
``` bash
"%USERPROFILE%\Desktop\sratoolkit\bin\prefetch.exe" --version
```
Then enter
``` bash
"%USERPROFILE%\Desktop\sratoolkit\bin\fasterq-dump.exe" --version
```
    
To download all 8 sequencing runs for GSM9221430 (RA_1), we'll use a Windows batch loop in Command Prompt. This will download each run into its own folder. The command will run *prefetch* eight times, once for each accession. Your folders will look like this:
C:\GEO_raw\GSE307297\sra\
├── SRR35257384\
├── SRR35257385\
├── SRR35257386\
├── SRR35257387\
├── SRR35257388\
├── SRR35257389\
├── SRR35257422\
└── SRR35257423

``` bash
for %R in (SRR35257384 SRR35257385 SRR35257386 SRR35257387 SRR35257388 SRR35257389 SRR35257422 SRR35257423) do "%USERPROFILE%\Desktop\sratoolkit\bin\prefetch.exe" %R -O "C:\GEO_raw\GSE307297\sra%R"
```
Once successful, we'll convert all eight downloaded SRA runs into FASTQ files using *fasterq-dump*. We'll save the FASTQ files in the fastq folder and use the temp folder for temporary files.

``` bash
for %R in (SRR35257384 SRR35257385 SRR35257386 SRR35257387 SRR35257388 SRR35257389 SRR35257422 SRR35257423) do "%USERPROFILE%\Desktop\sratoolkit\bin\fasterq-dump.exe" "C:\GEO_raw\GSE307297\sra%R%R.sra" --split-files -O "C:\GEO_raw\GSE307297\fastq" -t "C:\GEO_raw\GSE307297\temp"
```
To check the output files (this will provide a list of all the files that was created)
``` bash
dir "C:\GEO_raw\GSE307297\fastq"
#this will provide the number of files that exist
dir /b "C:\GEO_raw\GSE307297\fastq\*.fastq" | find /c /v ""
```
We observe that two fastq files have been extracted from each sra file. This is because the original data was produced from paired-end sequencing, which usually has both a Read1 file and Read2 file. `fastq-dump` has extracted the SRA file into two files, with suffix "\_1" for paired-end read 1 and "\_2" for paired-end read 2. I typically use the settings provided above for `fastq-dump` as my default settings.



Quality Control on FastQC
----------------------------------
FastQC is a separate program from SRA Toolkit. It generates a quality report for each FASTQ file. FastQC can be downloaded on <https://www.bioinformatics.babraham.ac.uk/projects/fastqc/>. 
``` bash
#this searches for the FastQC on the desktop
dir "%USERPROFILE%\Desktop\fastqc" /s /b | findstr /i "fastqc.bat"
#create a folder for the FastQC report
mkdir C:\GEO_raw\GSE307297\fastqc_reports
```
Note that FastQC is a Java-based program, therefore it needs to be installed before FastQC can generate reports <https://adoptium.net> Once installed, close the command prompt to allow it to refresh. 
``` bash
java -version
#Move into the FastQC folder
cd /d "C:\Users\John\Desktop\fastqc"
#Run all 16 FastQ files from the GSE307297 dataset (this finds every FastQ file in the FastQ folder, and runs each individually)
for %F in ("C:\GEO_raw\GSE307297\fastq\*.fastq") do java -Xmx250m -Dfastqc.output_dir="C:\GEO_raw\GSE307297\fastqc_reports" -cp ".;*" uk.ac.babraham.FastQC.FastQCApplication "%~F"
#Confirms that the reports were generated
dir /b "C:\GEO_raw\GSE307297\fastqc_reports\*_fastqc.html"
```

To open the folder, go on File Explorer and press the window key + E and at the address bar at the top of the window type this path in 
``` bash
C:\GEO_raw\GSE307297\fastqc_reports
```
Find a file ending in _fastqc.html, such as SRR35257385_2_fastqc.html, and double-click it. It should open in your web browser and display the FastQC report.
This link has a good reference of how to analyze the FastQC report. 
<https://docs.rcac.purdue.edu/lifesciences/guides/qc-for-genomics/> or <https://bioinfo.cd-genomics.com/quality-control-how-do-you-read-your-fastqc-results.html>


- R1 (_1.fastq) generally contains the cell barcode and UMI information in standard 10x 3′ gene-expression libraries. Be especially cautious. In standard 10x 3′ gene-expression libraries, R1 contains the cell barcode and UMI. Trimming or changing these positions can prevent Cell Ranger from correctly identifying cells and counting molecules.
- R2 (_2.fastq) generally contains the transcript-derived sequence used to identify genes. Adapter or quality trimming may sometimes be appropriate, but only after investigating the actual problem and checking the recommended workflow.
The exact read structure depends on the library chemistry. A red warning for base composition in R1 may therefore be expected. 

  We should not automatically trim or alter these reads, because doing so could damage the barcode and UMI information needed for cell identification and molecule counting.

The key principle: identify the cause → choose a justified correction → preserve the originals → rerun QC. A red FastQC result alone is not enough to determine which correction to apply.



| Module | What It Measures | Good | Bad | Common False Flags |
| --- | --- | --- | --- | --- |
| **Basic Statistics** | Read count, length, GC% | Consistent with expectations | Unexpected read count or length | Rarely triggers false flags |
| **Per Base Sequence Quality** | Phred scores at each position along the read | Q28+ across most positions; slight drop at 3' end is normal | Q < 20 across large portions of the read | Rarely false; always investigate |
| **Per Tile Sequence Quality** | Quality variation across flow cell tiles | Uniform blue heatmap | Hot spots (yellow/red patches) indicate flow cell problems | Rare |
| **Per Sequence Quality Scores** | Distribution of average quality per read | Single peak at Q30-Q37 | Bimodal distribution (two peaks)| Rare |
| **Per Base Sequence Content** | %A, %T, %G, %C at each position | Roughly equal A/T and G/C; flat lines | Strong bias throughout the read| First 10-15 bases often wobble due to random hexamer priming. This is normal |
| **Per Sequence GC Content**| Distribution of GC% across all reads | Smooth bell curve matching your organism | Bimodal curve = contamination from another organism | Rarely false; investigate if unexpected |
| **Per Base N Content** | Percentage of N (unknown) bases at each position | Near 0% everywhere | Spikes of N calls indicate sequencer problems | Rare |
| **Sequence Length Distribution** | Distribution of read lengths| All reads same length (pre-trimming) | Unexpected variation | After trimming, variation is expected and normal |
| **Sequence Duplication Levels**| How many reads appear more than once | Most reads unique (low duplication) | High duplication in WGS = low library complexity | RNA-seq: high duplication is normal (highly expressed genes). Amplicon: high duplication is expected.|
| **Overrepresented Sequences** | Sequences making up > 0.1% of reads | None or very few | Adapter sequences, rRNA contamination | RNA-seq: abundant transcripts trigger this. Not always a problem. |
|**Adapter Content** | Cumulative adapter contamination at each position | Near 0% throughout | Rising curves at read ends = adapter read-through | Rarely false; needs trimming if present |
| **Kmer Content** | Overrepresented short sequences at specific positions | No enrichment | Enrichment at specific positions = primer or adapter contamination | First few bases often enriched due to priming bias, usually normal |

When there are many samples, opening individual FastQC reports one by one is very tedious. **MultiQC** aggregates results from FastQC into a single interactive HTML report. To install MutliQC, you can use Python. Once it is confirmed that Python is installed, the following code can be typed on the command window. 

``` bash
#this command installs MultiQC and its required Python packages. The official installation instructions also support pip
python -m pip install multiqc
#Checks that MultiQC was installed
python -m multiqc --version
#Generates a MultiQC
python -m multiqc "C:\GEO_raw\GSE307297\fastqc_reports" -o "C:\GEO_raw\GSE307297\multiqc_report"
#Opens the report
explorer "C:\GEO_raw\GSE307297\multiqc_report"
```
Then open multiqc_report.html in your browser. 

Note that if some of the bases have low quality, especially towards the end of the read, they can be trimmed or truncated to remove low quality bases as well as adapter sequence. The rule of thumb for your 10x Genomics 3′ single-cell RNA-seq data is:
- R1 (Read 1): Generally, you do not trim it because it contains the cell barcode and UMI needed to identify cells and captured molecules.
- R2 (Read 2): You may trim it if necessary, for example, to remove confirmed adapter contamination or problematic low-quality terminal bases.Once the file is trimmed, run quality control again.  

1. Run FastQC and MultiQC
Assess read quality and identify unusual patterns.


2. Investigate warnings
Determine whether they reflect expected RNA-seq characteristics or a genuine technical problem.


3. Preprocess only when justified
Trim poor-quality bases or adapters if needed and compatible with the 10x library design. Don't trim simply to make the report green.


4. Run Cell Ranger
It processes the 10x reads, handles cell barcodes and UMIs, aligns transcript reads, and quantifies gene expression. It is not a universal fix for every possible sequencing or library-preparation problem.


Alignment and Quantification: Generating a Count Matrix
----------------------------------
Mapping or Alignment is a critical step in single-cell raw data processing. It involves determining the potential loci of origin for each sequenced fragment, such as the genomic or transcriptomic locations that closely match the read sequence. This step is essential for correctly assigning reads to their source regions.

Note that Cell Ranger runs on Linux, in the Ubuntu terminal check what version, how many CPU cores are available, the available memory (RAM) and the available disk space. Then confirm that Ubuntu can access your FASTQ files
``` bash
ls /mnt/c/GEO_raw/GSE307297/fastq
#check whether your D: drive is accessible:
ls /mnt/d
#check its available space from Ubuntu
df -h /mnt/d
check whether your FASTQ files are large and how much space they need
du -sh /mnt/c/GEO_raw/GSE307297/fastq
#check the available space on your Linux filesystem again
df -h /
```
Note that on computer although C:drive has some space, it is not enough to safely process this dataset here C:\	119 GB, however the D:\	3.0 TB has enough storage for the project files and Cell Ranger output, since the  FASTQ files alone occupy 438 GB. Cell Ranger also creates intermediate files, alignment data, and output matrices, so its storage needs will be substantially larger than just the input files. 10x Genomics lists 1.5 TB of free disk space as its general requirement for Cell Ranger v10.1, although actual requirements vary by dataset.


``` bash
#check whether D: is accessible and has enough room
ls -ld /mnt/d
#checks that Ubuntu can access your D: drive.
du -sh /mnt/c/GEO_raw/GSE307297/fastq
#Create your RA-ILD project folders on D:
mkdir -p /mnt/d/RAILD_CellRanger/{fastq,reference,output,software}
#Confirm the folders were created
ls -la /mnt/d/RAILD_CellRanger
```
We'll continue using Ubuntu and your existing files on C: for now, so you don't have to copy another 438 GB to D: yet.
``` bash
ls -lh /mnt/c/GEO_raw/GSE307297/fastq/*.fastq
```
Then on the GEO database, select SRA run selector and in the select section, download the total metadata. This will provide a SraRunTable.cvs. 


``` bash
#Prepare the software folder  
mkdir -p /mnt/d/RAILD_CellRanger/software
cd /mnt/d/RAILD_CellRanger/software
pwd
```

Then open the official Cell Ranger v10.0 download page. Find the Download Cell Ranger link and select the Linux version. We want the Linux .tar.gz software archive, on the website they provide a terminal command  (curl) that downloads the archive directly into the current Ubuntu folder. Once the download is complete check by 
``` bash
#Verify the archive
cd /mnt/d/RAILD_CellRanger/software
ls -lh cellranger-10.1.0.tar.gz
#Check the integrity
tar -tzf cellranger-10.1.0.tar.gz | head
#Extract the CellRanger
tar -xzf cellranger-10.1.0.tar.gz
#Checks CellRanger
ls -ld /mnt/d/RAILD_CellRanger/software/cellranger-10.1.0


#Makes CellRanger available on Ubuntu
/mnt/d/RAILD_CellRanger/software/cellranger-10.1.0/cellranger --version
#adding CellRanger to Path permanently
echo 'export PATH=/mnt/d/RAILD_CellRanger/software/cellranger-10.1.0:$PATH' >> ~/.bashrc
#Applying the change to Ubuntu
source ~/.bashrc
#verifying the command
which cellranger
cellranger --version
```


**Note:** that Cell Ranger needs two main inputs to process the RA-ILD 1 data: FASTQ files and a compatible reference transcriptome. The current standard human reference transcriptome for scRNA seq that is compatible with CellRanger (v9.0 or higher) is the human **GRCh38-2024-A** reference.
``` bash
#####Download the human reference transcriptome
#move into your reference folder in Ubuntu
cd /mnt/d/RAILD_CellRanger/reference
#Check
pwd
#Download Reference package (this will take a while)
curl -L -o refdata-gex-GRCh38-2024-A.tar.gz \
"https://cf.10xgenomics.com/supp/cell-exp/refdata-gex-GRCh38-2024-A.tar.gz"
#Check downloaded file
ls -lh refdata-gex-GRCh38-2024-A.tar.gz
#Check that the archive can be read
tar -tzf refdata-gex-GRCh38-2024-A.tar.gz | head
#Extracting (this will take a while)
tar -xzf refdata-gex-GRCh38-2024-A.tar.gz
#verify the reference folder
ls -ld refdata-gex-GRCh38-2024-A
#Check its contents
 ls refdata-gex-GRCh38-2024-A

#Inspect the reference metadata
cat /mnt/d/RAILD_CellRanger/reference/refdata-gex-GRCh38-2024-A/reference.json
#check that all eight FASTQ pairs exist (output should show 16 files) 
for r in SRR35257384 SRR35257385 SRR35257386 SRR35257387 SRR35257388 SRR35257389 SRR35257422 SRR35257423; do
  for read in 1 2; do
    f="/mnt/c/GEO_raw/GSE307297/fastq/${r}_${read}.fastq"
    if [ -s "$f" ]; then
      echo "OK   $r Read $read"
    else
      echo "MISSING or EMPTY   $r Read $read"
    fi
  done
done
#
head -n 8 /mnt/c/GEO_raw/GSE307297/fastq/SRR35257384_1.fastq
```


 CellRanger
| Files from CellRanger | Description | 
| --- | --- |
| 1.filtered_feature_bc_matrix/| Your main starting point Contains three files: barcodes.tsv.gz, features.tsv.gz and matrix.mtx.gz. What it's for: starting your Seurat analysis with barcodes Cell Ranger classified as cells. You'll use this to analyze gene expression, filter cells further, cluster them and identify plasma cells.|
| 2. raw_feature_bc_matrix/ |Contains a broader set of detected barcodes, including background barcodes that may not represent genuine cells. What it's for: specialized analyses that need the broader barcode set, such as reassessing cell calling or investigating background RNA. You generally don't need this as your default starting matrix. | 
| 3. web_summary.html| An interactive quality-control report. What it's for: checking whether the sequencing and processing look reasonable before moving into Seurat.| 
| 4.metrics_summary.csv| A summary of Cell Ranger metrics. What it's for: recording and comparing key metrics across samples.| 
| 5.possorted_genome_bam.bam and molecule_info.h5| These contain lower-level alignment and molecule information. What they're for: specialized downstream analyses, troubleshooting, or workflows that need read-level or molecule-level data. You don't normally need to load these to start a standard Seurat analysis.| 








