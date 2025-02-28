# Salus Spatial Workflow

This workflow performs cell filtering, clustering, differential expression analysis, marker visualization and compositional analysis on the provided matrix.

## Tips

* Both `sampleID` and `chipID` cannot use `_`, `.`, and spaces.
* If you modified `config/fqlist.csv` after `results` had come out, it is suggested to restart in another empty dir.

## Usage

### 1. Preparation

#### 1.1 Install `cuda-toolkit` (Version>=12.1) following the offical Nvidia Guide:

<https://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#package-manager-installation>.

#### 1.2 Unpack wfGEX.tgz

```bash
rm -fr ./wfGEX/
tar -xf wfGEX.tgz
cd wfGEX
```

### 2. Install Conda Envierment

#### 2.0 Clean `.condarc` contents during installing

```bash
mv ~/.condarc ~/moved.condarc
#rm ~/.condarc
micromamba config remove-key channels
micromamba config append channels pyg channels pytorch channels nvidia channels conda-forge channels bioconda 
micromamba config append channels salus channels galaxy001 channels https://software.repos.intel.com/python/conda/
#micromamba config remove channels defaults
micromamba config set channel_priority flexible
#micromamba config set auto_activate_base false
micromamba info
cat ~/.condarc
```

If you are not using `micromamba`, just modify `~/.condarc` to the contents below.

We suggest [Miniforge3](https://github.com/conda-forge/miniforge?tab=readme-ov-file#miniforge3) in case [micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html) is not available. [Miniconda](https://docs.anaconda.com/miniconda/) is not suggested as extra configuration on `channels` is required.

##### `~/.condarc` Contents

```yaml
channels:
  - pyg
  - pytorch
  - nvidia
  - conda-forge
  - bioconda
  - salus
  - galaxy001
  - https://software.repos.intel.com/python/conda/
  - nodefaults
channel_priority: flexible
#solver: libmamba
#auto_activate_base: false
```

##### Conda 镜像

国内用户通常需要使用镜像加速文件下载。可以参考清华大学的设置说明：<https://mirror.tuna.tsinghua.edu.cn/help/anaconda/>。

```bash
conda config --set custom_channels.conda-forge https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/
conda config --set custom_channels.bioconda https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/
conda config --set custom_channels.pytorch https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/
```

#### 2.1 Setup Conda Envirement *salus*

```bash
echo ${MAMBA_EXE:=conda env}

$MAMBA_EXE create -n salus -f environmentGPU.yml
```

#### 2.2 Optional: install R packages (Require International Internet connection)

```bash
conda env update -n salus -f environmentR.yml

### OR ###

micromamba install -n salus -f environmentR.yml
```

#### 2.3 Restore `$MAMBA_EXE` value & `~/.condarc` contents after installing

```bash
unset MAMBA_EXE
eval `grep MAMBA_EXE=  ~/.bashrc`

mv ~/moved.condarc ~/.condarc
```

### 3. Build STAR2 Reference

If you already have STAR(version 2.7) reference files, you can ignore this part and go to the next step.

```bash
export INSTALLPREFIX=/share/pub/Salus/salusts

mkdir -p $INSTALLPREFIX/ref
curl -O "https://cf.10xgenomics.com/supp/spatial-exp/refdata-gex-GRCh38-2020-A.tar.gz" --output-dir $INSTALLPREFIX/
curl -O "https://cf.10xgenomics.com/supp/spatial-exp/refdata-gex-mm10-2020-A.tar.gz" --output-dir $INSTALLPREFIX/
tar xvf $INSTALLPREFIX/refdata-gex-GRCh38-2020-A.tar.gz -C $INSTALLPREFIX/ref/
tar xvf $INSTALLPREFIX/refdata-gex-mm10-2020-A.tar.gz -C $INSTALLPREFIX/ref/

echo ${MAMBA_EXE:=conda}

$MAMBA_EXE activate salus
cd $INSTALLPREFIX/ref/refdata-gex-GRCh38-2020-A
STAR --runMode genomeGenerate --runThreadN 24 --genomeDir read150 --genomeFastaFiles fasta/genome.fa --sjdbGTFfile genes/genes.gtf --sjdbOverhang 149 >read150.log
cd $INSTALLPREFIX/ref/refdata-gex-mm10-2020-A
STAR --runMode genomeGenerate --runThreadN 24 --genomeDir read150 --genomeFastaFiles fasta/genome.fa --sjdbGTFfile genes/genes.gtf --sjdbOverhang 149 >read150.log

ls -l $INSTALLPREFIX/ref/refdata-gex-GRCh38-2020-A/read150
ls -l $INSTALLPREFIX/ref/refdata-gex-mm10-2020-A/read150
```

### 4. Prepare External Data

```bash
export INSTALLPREFIX=/share/pub/Salus/salusts

mkdir -p $INSTALLPREFIX/segment-anything
curl https://dl.fbaipublicfiles.com/segment_anything/sam_vit_h_4b8939.pth --remote-name --create-dirs --output-dir $INSTALLPREFIX/segment-anything/
```

### 5. Modify `workflow/Snakefile` Contents

`export INSTALLPREFIX=/share/pub/Salus/salusts`

```python
StarRefdict = {
    'mouse': '/share/pub/Salus/salusts/ref/10Xrefdata-gex-mm10-2020-A/read150',
    'human': '/share/pub/Salus/salusts/ref/10Xrefdata-gex-GRCh38-2020-A/read150',
}
mtGeneLstdict = {
    'human': '/dev/null',
    'mouse': '/dev/null',
}
usedModels = {
    'segment_anything': '/share/pub/Salus/salusts/segment-anything/sam_vit_h_4b8939.pth'
}
```

### 6. Install the SaluSTS workflow

```bash
export INSTALLPATH=/share/pub/Salus/salusts/v280
rm -fr $INSTALLPATH/workflow
mkdir -p $INSTALLPATH
cp -a config $INSTALLPATH/
cp -a workflow $INSTALLPATH/
ls -l $INSTALLPATH/
```

### 7. Use the SaluSTS workflow

#### with Conda Envirement

```bash
export INSTALLPATH=/share/pub/Salus/salusts/v280
cd /your/workdir/
mkdir job01
cd job01
ln -s $INSTALLPATH/workflow .
cp -a $INSTALLPATH/config .
vim config/fqlist.csv
vim config/config.yaml
tmux
echo ${MAMBA_EXE:=conda}

$MAMBA_EXE activate salus
snakemake -c32 -p --dry-run	# `--dry-run` == `-n`
snakemake -c32 -p >r1.std 2>r1.log &

# ManualHEmask Step 1
ls -l results/ManualHEmask/pre_*/*.tif  # Download HEimg_sub.tif(s) and GeExmap.tif(s)
# ManualHEmask Step 3
ls -l results/ManualHEmask/*.mask.npy   # Upload your mask.npy files to given positions.
# ManualHEmask Step 4
mv results.done results.done.0          # remove finished state tag
vim config/config.yaml                  # set ReadManualHEmask to True
snakemake -c32 -p >r2.std 2>r2.log &
```

#### with Docker

```bash
# If you can read docker documents
# Pull form DockerHub, minimal tagged version is v280R
#docker image pull galaxy001/salusts:v281R
#docker image tag galaxy001/salusts:v281R galaxy001/salusts:latest

# If you are too busy to read
docker image pull galaxy001/salusts:latest

# Run as Host root
(time docker run --rm --volume ${PWD}:/app --volume /share:/share galaxy001/salusts snakemake --cores 64 -p )>r1.std 2>r1.log &

# Run as Host user
(time docker run --rm -u $(id -u ${USER}):$(id -g ${USER}) --volume ${PWD}:/app --volume /share:/share galaxy001/salusts snakemake --cores 64 -p )>r1.std 2>r1.log &
```

## Configure files

### config/fqlist.csv

* seqType : 用于标记输入fastq序列文件属于芯片测序文件（fst）还是用户实验测序文件（snd）；
* fqPath : 路径下存放测序文件，必须为fastq.gz或fq.gz结尾。snd行输入路径同时包含sndBC与sndNA两测序文件；
* chipZone : 记录实验时所使用的孔位，将用于后续分析的裁剪。注意在fstBC测序文件所在行不需输入；
* sndBarcodePrimer : 记录用户建库实验后文库的测序引物，pe或se。fstBC测序文件所在行不需输入；se仅限内部测试，普通试剂盒都是pe模式。
* sampleID : 对应fstBC所在行的芯片编码号，及sndBC和sndNA所在行的用户自定义样品名。注意样品名中不可有空格、小数点和下划线。

### config/config.yaml

```yaml
fqs: config/fqlist.csv

ChipType: Dual   # Solo, DualUpper, DualLower, Dual
STARref: custom  # human or mouse; custom
umilength: 10
PEmode: cut  # normal or cut
BCUMI2pass: True  # Set True to use Salus barcode correction tool over the one from STAR.
useSTARsort: True  # False for sorting by SAMTOOLS(less Mem and opened files required).
TissueSegment: True # Use SegmentAnything to extract tissue region. Set False also disable HTML report.
TissueSegmentBinSize: 100 # For both SegmentAnything and manual. Must exists in pseudo_cell.binsize
HEtifPath: ./  # For multiple samples, store the corresponding images named as {sampleID}.tif in {HEtifPath}. The program recognizes them automatically.
TissueSegmentHE: False  # manual tissue region extraction step 1. Will read {HEtifPath}/{sampleID}.tif
ReadManualHEmask: False # step 4, read mask file results/ManualHEmask/{sampleID}_{chipZone}.mask.npy
STAGATE: False

ref:
  custom:
    star: /share/data/reference/bee/Amel_HAv31n_sjdbOverhang150
    mtgenes: /dev/null

sndRNA:
  EnableSoloVelocyto: yes    # Required for cellbin results.
  soloStrandFlag: 1 # 1 = Forward, 2 = Reverse, 3 = Unstranded

pseudo_cell:
  binsize:
    - 400   # 100um x 100um
    - 100   # 25um x 25 um, please keep bin100
    - 40    # 10um x 10 um

withR: False  # Require The Optional R packages
withClean: 0  # 0 = Auto, 1 = True, -1 = False
```

## Results

### Results Directory Scaffolds

```
./results/
├── filtered_fastq
├── fstCoord_sndBCs
├── estimate_tissueBoundary
├── merge
├── saluscbmap
├── star
│   └── PE-Test1-Sample3_B2
│       └── Solo.out
│           ├── GeneFull_Ex50pAS
│           ├── Gene
│           └── Velocyto
├── saturation
├── qc_stat
├── simple_grids
├── segmentation
├── ManualHEmask
│   ├── pre_SE-Yang4-Pin3_C1
│   └── pre_PE-Test1-Sample3_B2
├── html
└── htmlHE
```

See [Results.md](Results.md) for details.
