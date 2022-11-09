# Segregation: A repository for segregation analysis
This repository includes module and pipeline for segregation analysis, which can be used in PC or HPC. 

## Contents
-  [Segregation Analysis](#segregation-Analysis)
-  [How to run](#how-to-run)
   - [seganalyis](#workflow)
   - [segrun.svn](#scrnaboxsvn)
- [References](#references)


## Segregation Analysis
Segregation is a process to explore the genetic variant in a sample of seguence data. This pipeline counts the number of affecteds and nonaffecteds with variant, with homozygous variant, with no variant, and with no call. It gets those counts both in-family and globally. Also we also get the breakdown of not just variants, but also the breakdown of alleles in each. To achive the segregation, one needs a pedigree file with six columns: `familyid`, `individualid`, `parentalid`, `maternalid`, `sex`{1:male; 2:female, 0:unknown}, and `phenotype`={1: control (unaffected), 2: proband(affected), -9:missing}. And the genetic data must be in the `vcf` format.

## How to run
The segregation can do done using the  `seganalyis` module in python, if you have access to HPC, you can automate it using segrun.svn. 

### seganalyis
`seganalyis` is a python module developed to run segregation, 

|<img src="https://raw.githubusercontent.com/neurobioinfo/segregation/main/segpy.png" width="500" height="400"/>|
|:--:|
| _segpy workflow_ |


The following steps show how to run the segregation pipeline.

#### Step 1: Run Spark 
First activate Spark on your system 
```
export SPARK_HOME=$HOME/spark-3.1.2-bin-hadoop3.2
export SPARK_LOG_DIR=$HOME/temp
module load java/11.0.2
cd ${SPARK_HOME}; ./sbin/start-master.sh
```

#### Step 2: Generate annotated file 
Run [VEP](https://useast.ensembl.org/info/docs/tools/vep/script/vep_tutorial.html) to generate the annotated file.   

#### Step 3:  Create table matrix
Next, initialize the hail and import your vcf file and write it as a matrix table, the matrix table is a data structure to present the genetic data as a matrix. In the below, we import the vcf file and write it as `MatrixTable`, then read your matrix table.
For the tutorial, we add data to test the pipeline [https://github.com/The-Neuro-Bioinformatics-Core/seganalysis/test]. The pipeline is explained using this dataset. 

The following code imports VCF as a MatrixTable: 
```
import sys
import pandas as pd 
import hail as hl
hl.import_vcf('~/test/data/testseg.VEP.vcf',force=True,reference_genome='GRCh38',array_elements_required=False).write('~/test/output/testseg.mt', overwrite=True)
mt = hl.read_matrix_table('~/test/output/testseg.mt')
```

#### Step 4: Run the module
Run the following codes to generate the segregation. 
```
from seganalysis import seg
ped=pd.read_csv('~/test/data/testseg.ped'.ped',sep='\t')
destfolder= '~/test/output/'
vcffile='~/test/data/testseg.VEP.vcf'
seg.segrun(mt,ped,outfolder,hl,vcffile)    
```
It generates two files `header.txt` and `finalseg.csv` in the  `destfolder`; `header.txt`  includes the header of information in `finalseg.csv`. The  
output  of `finalseg.csv` can be categorized to  1) locus and alleles, 2) CSQ, 3) Global- Non-Affected 4) Global-Affected,  5) Family, 6) Family-Affected 7) Family - Non-affected.  

##### locus and alleles
locus: chromosome <br/>
alleles:  a variant form of a gene
##### CSQ
VEP put all the requested information in infront CSQ, running  `seg.segrun()` split CSQ to columns.  
##### Global - Non-Affected
glb_naf_wild:  Global - Non-Affecteds, wildtype<br/>
glb_naf_ncl:     Global - Non-Affecteds, no call  <br/>   
glb_naf_vrt:     Global - Non-Affecteds, with variant    <br/>
glb_naf_homv:    Global - Non-Affecteds, homozygous for ALT allele<br/>
glb_naf_altaf:   Global - Non-Affecteds, ALT allele frequency   <br/>
##### Global - Affected
glb_aff_wild: Global - Affecteds, wildtype <br/>
glb_aff_ncl:     Global - Affecteds, no call    <br/> 
glb_aff_vrt:     Global - Affecteds, with variant  <br/>
glb_aff_homv:    Global - Affecteds, homozygous for ALT allele<br/>
glb_aff_altaf:   Global - Affecteds, ALT allele frequency   <br/>
##### Family
{famid}_wild: Family - Affecteds: wildtype <br/>
{famid}_ncl: Family - Affecteds: no call<br/>
{famid}_vrt: Family - Affecteds: with variant<br/>
{famid}_homv: Family - Affecteds: homozygous for ALT allele<br/>
##### Family - Affected
{famid}_wild_aff: Family - Affecteds: wildtype <br/>
{famid}_ncl_aff: Family - Affecteds: no call<br/>
{famid}_vrt_aff: Family - Affecteds: with variant<br/>
{famid}_homv_aff: Family - Affecteds: homozygous for ALT allele<br/>
##### Family - Nonaffected   
{famid}_wild_naf: Family - Nonaffecteds: wildtype <br/>
{famid}_ncl_naf: Family - Nonaffecteds: no call<br/>
{famid}_vrt_naf: Family - Nonaffecteds: with variant<br/>
{famid}_homv_naf: Family - Nonaffecteds: homozygous for ALT allele<br/>

#### Step 5: Parsing
If you want to select a subset of header, you can define them in a file, and running
the following codes.  
```
from seganalysis import parser
header_need='~/test/data/header_need.txt'
parser.sub_seg(outfolder, header_need)  
```

#### Step 6  Shut down spark  
Do not forget to deactivate environment and stop the spark: 
```
cd ${SPARK_HOME}; ./sbin/stop-master.sh
```


### segrun.svn
`segrun.svn` is a pipeline (shield) developed to run step 1 to step 3 under HPC system, the pipeline has been using under [Beluga](https://docs.alliancecan.ca/wiki/B%C3%A9luga),  the detail of using it is discussed in [segrun.svn](https://github.com/neurobioinfo/segregation/tree/main/segrun.svn)


|<img src="https://raw.githubusercontent.com/neurobioinfo/segregation/main/segrun.png" width="200" height="400"/>|
|:--:|
| _segrun workflow_ |


To run the pipepline, you need 1) path of the pipeline (PIPELINE_HOME), 2) Working directory , 3) VCF, and 4) PED file
```
export PIPELINE_HOME=/lustre03/project/6004655/COMMUN/runs/samamiri/general_files/hail/segrun.svn
PWD=/lustre03/project/6004655/COMMUN/runs/samamiri/general_files/hail/practiceiPSC
VCF=/lustre03/project/6004655/COMMUN/runs/samamiri/general_files/hail/practiceiPSC/VCF_xy.vcf
PED=/lustre03/project/6004655/COMMUN/runs/samamiri/general_files/hail/practiceiPSC/iPSC_PBMC.ped
```

#### Step1: Setup
First run runs the following code to setup the pipeline, you can change the the parameters in ${PWD}/job_output/segrun.config.ini
```
sh $PIPELINE_HOME/launch_pipeline.segrun.sh \
-d ${PWD} \
--steps 0
```

#### Step 2:  Create table matrix
The following code, import VCF as a MatrixTable, 
```
sh $PIPELINE_HOME/launch_pipeline.segrun.sh \
-d ${PWD} \
--steps 1 \
--vcf ${VCF}
```

#### Step 3: Run the segregation 
```
sh $PIPELINE_HOME/launch_pipeline.segrun.sh \
-d ${PWD} \
--steps 2 \
--vcf ${VCF} \
--ped ${PED}  
```

#### Step 4: Clean final data
```
sh $PIPELINE_HOME/launch_pipeline.segrun.sh \
-d ${PWD} \
--steps 3 
```


## References

## Contributing
This is an early version, any contribute or suggestion is appreciated, you can directly contact with [Saeid Amiri](https://github.com/saeidamiri1) or [Dan Spiegelman](https://github.com/danspiegelman).

## Citation
Amiri, S., Spiegelman, D., & Farhan, S. (2022). segregation: A pipeline for segregation analysis (Version 0.1.0) [Computer software]. https://github.com/neurobioinfo/segregation

## Changelog
Every release is documented on the [GitHub Releases page](https://github.com/neurobioinfo/segregation/releases).

## License
This project is licensed under the MIT License - see the [LICENSE.md](https://github.com/neurobioinfo/segregation/blob/main/LICENSE) file for details

## Acknowledgement
The pipeline is done as a project by Neuro Bioinformatics Core, it is written by [Saeid Amiri](https://github.com/saeidamiri1) with associate of Dan Spiegelman and Sali Farhan. 

## Todo


  **[⬆ back to top](#contents)**
