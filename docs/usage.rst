Use Pegasus as a command line tool
---------------------------------------

Pegasus can be used as a command line tool. Type::

	pegasus -h

to see the help information::

	Usage:
		pegasus <command> [<args>...]
		pegasus -h | --help
		pegasus -v | --version

``pegasus`` has 9 sub-commands in 6 groups.

* Preprocessing:

	aggregate_matrix
		Aggregate sample count matrices into a single count matrix. It also enables users to import metadata into the count matrix.

* Demultiplexing:

	demuxEM
		Demultiplex cells/nuclei based on DNA barcodes for cell-hashing and nuclei-hashing data.

* Analyzing:

	cluster
		Perform first-pass analysis using the count matrix generated from 'aggregate_matrix'. This subcommand could perform low quality cell filtration, batch correction, variable gene selection, dimension reduction, diffusion map calculation, graph-based clustering, visualization. The final results will be written into zarr-formatted file, or h5ad file, which Seurat could load.

	de_analysis
		Detect markers for each cluster by performing differential expression analysis per cluster (within cluster vs. outside cluster). DE tests include Welch's t-test, Fisher's exact test, Mann-Whitney U test. It can also calculate AUROC values for each gene.

	find_markers
		Find markers for each cluster by training classifiers using LightGBM.

	annotate_cluster
		This subcommand is used to automatically annotate cell types for each cluster based on existing markers. Currently, it works for human/mouse immune/brain cells, etc.

* Plotting:

	plot
		Make static plots, which includes plotting tSNE, UMAP, and FLE embeddings by cluster labels and different groups.

* Web-based visualization:

	scp_output
		Generate output files for single cell portal.

* MISC:

	check_indexes
		Check CITE-Seq/hashing indexes to avoid index collision.

---------------------------------


Quick guide
^^^^^^^^^^^

Suppose you have ``example.csv`` ready with the following contents::

	Sample,Source,Platform,Donor,Reference,Location
	sample_1,bone_marrow,NextSeq,1,GRCh38,/my_dir/sample_1/raw_feature_bc_matrices.h5
	sample_2,bone_marrow,NextSeq,2,GRCh38,/my_dir/sample_2/raw_feature_bc_matrices.h5
	sample_3,pbmc,NextSeq,1,GRCh38,/my_dir/sample_3/raw_gene_bc_matrices_h5.h5
	sample_4,pbmc,NextSeq,2,GRCh38,/my_dir/sample_4/raw_gene_bc_matrices_h5.h5

You want to analyze all four samples but correct batch effects for bone marrow and pbmc samples separately. You can run the following commands::

	pegasus aggregate_matrix --attributes Source,Platform,Donor example.csv example.aggr
	pegasus cluster -p 20 --correct-batch-effect --batch-group-by Source --louvain --umap example.aggr.zarr.zip example
	pegasus de_analysis -p 20 --labels louvain_labels example.zarr.zip example.de.xlsx
	pegasus annotate_cluster example.zarr.zip example.anno.txt
	pegasus plot compo --groupby louvain_labels --condition Donor example.zarr.zip example.composition.pdf
	pegasus plot scatter --basis umap --attributes louvain_labels,Donor example.zarr.zip example.umap.pdf

The above analysis will give you UMAP embedding and Louvain cluster labels in ``example.zarr.zip``, along with differential expression analysis
result stored in ``example.de.xlsx``, and putative cluster-specific cell type annotation stored in ``example.anno.txt``.
You can investigate donor-specific effects by looking at ``example.composition.pdf``.
``example.umap.pdf`` plotted UMAP colored by louvain_labels and Donor info side-by-side.


---------------------------------


``pegasus aggregate_matrix``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The first step for single cell analysis is to generate one count matrix from cellranger's channel-specific count matrices. ``pegasus aggregate_matrix`` allows aggregating arbitrary matrices with the help of a *CSV* file.

Type::

	pegasus aggregate_matrix -h

to see the usage information::

	Usage:
		pegasus aggregate_matrix <csv_file> <output_name> [--restriction <restriction>... --attributes <attributes> --default-reference <reference> --select-only-singlets --min-genes <number>]
		pegasus aggregate_matrix -h

* Arguments:

	csv_file
	csv_file should contains at least 2 columns — ``Sample``, sample name; ``Location``, file that contains the count matrices (e.g. ``filtered_gene_bc_matrices_h5.h5``), and merges matrices from the same genome together.
	If multi-modality exists, a third Modality column might be required. The csv_file can optionally contain two columns - ``nUMI`` and ``nGene``.
	These two columns define minimum number of UMIs and genes for cell selection for each sample. The values in these two columns overwrite the ``--min-genes`` and ``--min-umis`` options in command.
	See below for an example::

			Sample,Source,Platform,Donor,Reference,Location
 			sample_1,bone_marrow,NextSeq,1,GRCh38,/my_dir/sample_1/raw_feature_bc_matrices.h5
			sample_2,bone_marrow,NextSeq,2,GRCh38,/my_dir/sample_2/raw_feature_bc_matrices.h5
			sample_3,pbmc,NextSeq,1,GRCh38,/my_dir/sample_3/raw_gene_bc_matrices_h5.h5
			sample_4,pbmc,NextSeq,2,GRCh38,/my_dir/sample_4/raw_gene_bc_matrices_h5.h5

	output_name
		The output file name.

* Options:

	-\-restriction <restriction>...
		Select channels that satisfy all restrictions. Each restriction takes the format of name:value,...,value or name:~value,..,value, where ~ refers to not. You can specifiy multiple restrictions by setting this option multiple times.

	-\-attributes <attributes>
		Specify a comma-separated list of outputted attributes. These attributes should be column names in the csv file.

	-\-default-reference <reference>
		If sample count matrix is in either DGE, mtx, csv, tsv or loom format and there is no Reference column in the csv_file, use <reference> as the reference.

	-\-select-only-singlets
		If we have demultiplexed data, turning on this option will make pegasus only include barcodes that are predicted as singlets.

	-\-remap-singlets <remap_string>
		Remap singlet names using <remap_string>, where <remap_string> takes the format "new_name_i:old_name_1,old_name_2;new_name_ii:old_name_3;...". For example, if we hashed 5 libraries from 3 samples sample1_lib1, sample1_lib2, sample2_lib1, sample2_lib2 and sample3, we can remap them to 3 samples using this string: "sample1:sample1_lib1,sample1_lib2;sample2:sample2_lib1,sample2_lib2". In this way, the new singlet names will be in metadata field with key 'assignment', while the old names will be kept in metadata field with key 'assignment.orig'.

	-\-subset-singlets <subset_string>
		If select singlets, only select singlets in the <subset_string>, which takes the format "name1,name2,...". Note that if --remap-singlets is specified, subsetting happens after remapping. For example, we can only select singlets from sampe 1 and 3 using "sample1,sample3".

	-\-min-genes <number>
		Only keep barcodes with at least <ngene> expressed genes.

	-\-max-genes <number>
		Only keep cells with less than <number> of genes.

	-\-min-umis <number>
		Only keep cells with at least <number> of UMIs.

	-\-max-umis <number>
		Only keep cells with less than <number> of UMIs.

	-\-mito-prefix <prefix>
		Prefix for mitochondrial genes. If multiple prefixes are provided, separate them by comma (e.g. "MT-,mt-").

	-\-percent-mito <percent>
		Only keep cells with mitochondrial percent less than <percent>%. Only when both mito_prefix and percent_mito set, the mitochondrial filter will be triggered.

	-\-no-append-sample-name
		Turn this option on if you do not want to append sample name in front of each sample's barcode (concatenated using '-').

	\-h, -\-help
		Print out help information.

* Outputs:

	output_name.zarr.zip
		A zipped Zarr file containing aggregated data.

* Examples::

	pegasus aggregate_matrix --restriction Source:BM,CB --restriction Individual:1-8 --attributes Source,Platform Manton_count_matrix.csv aggr_data


---------------------------------

``pegasus demuxEM``
^^^^^^^^^^^^^^^^^^^^^

Demultiplex cell-hashing/nucleus-hashing data.

Type::

	pegasus demuxEM -h

to see the usage information::

	Usage:
  		pegasus demuxEM [options] <input_raw_gene_bc_matrices_h5> <input_hto_csv_file> <output_name>
  		pegasus demuxEM -h | --help
  		pegasus demuxEM -v | --version

* Arguments:

	input_raw_gene_bc_matrices_h5
		Input raw RNA expression matrix in 10x hdf5 format. It is important to feed raw (unfiltered) count matrix, as demuxEM uses it to estimate the background information.

	input_hto_csv_file
		Input HTO (antibody tag) count matrix in CSV format.

	output_name
		Output name. All outputs will use it as the prefix.

* Options:

	\-p <number>, -\-threads <number>
		Number of threads. [default: 1]

	-\-genome <genome>
		Reference genome name. If not provided, we will infer it from the expression matrix file.

	-\-alpha-on-samples <alpha>
		The Dirichlet prior concentration parameter (alpha) on samples. An alpha value < 1.0 will make the prior sparse. [default: 0.0]

	-\-min-num-genes <number>
		We only demultiplex cells/nuclei with at least <number> of expressed genes. [default: 100]

	-\-min-num-umis <number>
		We only demultiplex cells/nuclei with at least <number> of UMIs. [default: 100]

	-\-min-signal-hashtag <count>
		Any cell/nucleus with less than <count> hashtags from the signal will be marked as unknown. [default: 10.0]

	-\-random-state <seed>
		The random seed used in the KMeans algorithm to separate empty ADT droplets from others. [default: 0]

	-\-generate-diagnostic-plots
		Generate a series of diagnostic plots, including the background/signal between HTO counts, estimated background probabilities, HTO distributions of cells and non-cells etc.

	-\-generate-gender-plot <genes>
		Generate violin plots using gender-specific genes (e.g. Xist). <gene> is a comma-separated list of gene names.

	-v, -\-version
		Show DemuxEM version.

	-h, -\-help
		Print out help information.

* Outputs:

	output_name_demux.zarr.zip
		RNA expression matrix with demultiplexed sample identities in Zarr format.

	output_name.out.demuxEM.zarr.zip
		DemuxEM-calculated results in Zarr format, containing two datasets, one for HTO and one for RNA.

	output_name.ambient_hashtag.hist.pdf
		Optional output. A histogram plot depicting hashtag distributions of empty droplets and non-empty droplets.

	output_name.background_probabilities.bar.pdf
		Optional output. A bar plot visualizing the estimated hashtag background probability distribution.

	output_name.real_content.hist.pdf
		Optional output. A histogram plot depicting hashtag distributions of not-real-cells and real-cells as defined by total number of expressed genes in the RNA assay.

	output_name.rna_demux.hist.pdf
		Optional output. A histogram plot depicting RNA UMI distribution for singlets, doublets and unknown cells.

	output_name.gene_name.violin.pdf
		Optional outputs. Violin plots depicting gender-specific gene expression across samples. We can have multiple plots if a gene list is provided in '--generate-gender-plot' option.

* Examples::

	pegasus demuxEM -p 8 --generate-diagnostic-plots sample_raw_gene_bc_matrices.h5 sample_hto.csv sample_output

---------------------------------

``pegasus cluster``
^^^^^^^^^^^^^^^^^^^

Once we collected the count matrix in 10x (``example_10x.h5``) or Zarr (``example.zarr.zip``) format, we can perform single cell analysis using ``pegasus cluster``.

Type::

	pegasus cluster -h

to see the usage information::

	Usage:
		pegasus cluster [options] <input_file> <output_name>
		pegasus cluster -h

* Arguments:

	input_file
		Input file in either 'zarr', 'h5ad', 'loom', '10x', 'mtx', 'csv', 'tsv' or 'fcs' format. If first-pass analysis has been performed, but you want to run some additional analysis, you could also pass a zarr-formatted file.

	output_name
		Output file name. All outputs will use it as the prefix.

* Options:

	\-p <number>, -\-threads <number>
		Number of threads. [default: 1]

	-\-processed
		Input file is processed. Assume quality control, data normalization and log transformation, highly variable gene selection, batch correction/PCA and kNN graph building is done.

  	-\-channel <channel_attr>
		Use <channel_attr> to create a 'Channel' column metadata field. All cells within a channel are assumed to come from a same batch.

	-\-black-list <black_list>
		Cell barcode attributes in black list will be popped out. Format is "attr1,attr2,...,attrn".

	-\-select-singlets
		Only select DemuxEM-predicted singlets for analysis.

	-\-remap-singlets <remap_string>
		Remap singlet names using <remap_string>, where <remap_string> takes the format "new_name_i:old_name_1,old_name_2;new_name_ii:old_name_3;...". For example, if we hashed 5 libraries from 3 samples sample1_lib1, sample1_lib2, sample2_lib1, sample2_lib2 and sample3, we can remap them to 3 samples using this string: "sample1:sample1_lib1,sample1_lib2;sample2:sample2_lib1,sample2_lib2". In this way, the new singlet names will be in metadata field with key 'assignment', while the old names will be kept in metadata field with key 'assignment.orig'.

	-\-subset-singlets <subset_string>
		If select singlets, only select singlets in the <subset_string>, which takes the format "name1,name2,...". Note that if --remap-singlets is specified, subsetting happens after remapping. For example, we can only select singlets from sampe 1 and 3 using "sample1,sample3".

	-\-genome <genome_name>
		If sample count matrix is in either DGE, mtx, csv, tsv or loom format, use <genome_name> as the genome reference name.

	-\-focus <keys>
		Focus analysis on Unimodal data with <keys>. <keys> is a comma-separated list of keys. If None, the self._selected will be the focused one.

	-\-append <key>
		 Append Unimodal data <key> to any <keys> in ``--focus``.

	-\-output-loom
	 	Output loom-formatted file.

	-\-output-h5ad
		Output h5ad-formatted file.

  	-\-min-genes <number>
		Only keep cells with at least <number> of genes. [default: 500]

	-\-max-genes <number>
		Only keep cells with less than <number> of genes. [default: 6000]

	-\-min-umis <number>
		Only keep cells with at least <number> of UMIs.

	-\-max-umis <number>
		Only keep cells with less than <number> of UMIs.

	-\-mito-prefix <prefix>
		Prefix for mitochondrial genes. Can provide multiple prefixes for multiple organisms (e.g. "MT-" means to use "MT-", "GRCh38:MT-,mm10:mt-,MT-" means to use "MT-" for GRCh38, "mt-" for mm10 and "MT-" for all other organisms). [default: GRCh38:MT-,mm10:mt-,MT-]

	-\-percent-mito <ratio>
		Only keep cells with mitochondrial percent less than <percent>%. [default: 20.0]

	-\-gene-percent-cells <ratio>
		Only use genes that are expressed in at least <percent>% of cells to select variable genes. [default: 0.05]

	-\-output-filtration-results
		Output filtration results as a spreadsheet.

	-\-plot-filtration-results
		Plot filtration results as PDF files.

	-\-plot-filtration-figsize <figsize>
		Figure size for filtration plots. <figsize> is a comma-separated list of two numbers, the width and height of the figure (e.g. 6,4).

	-\-min-genes-before-filtration <number>
		If raw data matrix is input, empty barcodes will dominate pre-filtration statistics. To avoid this, for raw data matrix, only consider barcodes with at lease <number> genes for pre-filtration condition. [default: 100]

	-\-counts-per-cell-after <number>
		Total counts per cell after normalization. [default: 1e5]

	-\-select-hvf-flavor <flavor>
		Highly variable feature selection method. <flavor> can be 'pegasus' or 'Seurat'. [default: pegasus]

	-\-select-hvf-ngenes <nfeatures>
		Select top <nfeatures> highly variable features. If <flavor> is 'Seurat' and <ngenes> is 'None', select HVGs with z-score cutoff at 0.5. [default: 2000]

	-\-no-select-hvf
		Do not select highly variable features.

	-\-plot-hvf
		Plot highly variable feature selection.

	-\-correct-batch-effect
		Correct for batch effects.

	-\-correction-method <method>
		Batch correction method, can be either 'L/S' for location/scale adjustment algorithm (Li and Wong. The analysis of Gene Expression Data 2003), 'harmony' for Harmony (Korsunsky et al. Nature Methods 2019), 'scanorama' for Scanorama (Hie et al. Nature Biotechnology 2019) or 'inmf' for integrative NMF (Yang and Michailidis Bioinformatics 2016, Welch et al. Cell 2019, Gao et al. Natuer Biotechnology 2021) [default: harmony]

	-\-batch-group-by <expression>
		Batch correction assumes the differences in gene expression between channels are due to batch effects. However, in many cases, we know that channels can be partitioned into several groups and each group is biologically different from others. In this case, we will only perform batch correction for channels within each group. This option defines the groups. If <expression> is None, we assume all channels are from one group. Otherwise, groups are defined according to <expression>. <expression> takes the form of either 'attr', or 'attr1+attr2+...+attrn', or 'attr=value11,...,value1n_1;value21,...,value2n_2;...;valuem1,...,valuemn_m'. In the first form, 'attr' should be an existing sample attribute, and groups are defined by 'attr'. In the second form, 'attr1',...,'attrn' are n existing sample attributes and groups are defined by the Cartesian product of these n attributes. In the last form, there will be m + 1 groups. A cell belongs to group i (i > 0) if and only if its sample attribute 'attr' has a value among valuei1,...,valuein_i. A cell belongs to group 0 if it does not belong to any other groups.

	-\-harmony-nclusters <nclusters>
		Number of clusters used for Harmony batch correction.

	-\-inmf-lambda <lambda>
		Coefficient of regularization for iNMF. [default: 5.0]

	-\-random-state <seed>
		Random number generator seed. [default: 0]

	-\-temp-folder <temp_folder>
		Joblib temporary folder for memmapping numpy arrays.

	-\-calc-signature-scores <sig_list>
		Calculate signature scores for gene sets in <sig_list>. <sig_list> is a comma-separated list of strings. Each string should either be a <GMT_file> or one of 'cell_cycle_human', 'cell_cycle_mouse', 'gender_human', 'gender_mouse', 'mitochondrial_genes_human', 'mitochondrial_genes_mouse', 'ribosomal_genes_human' and 'ribosomal_genes_mouse'.

	-\-pca-n <number>
		Number of principal components. [default: 50]

	-\-nmf
		Compute nonnegative matrix factorization (NMF) on highly variable features.

	-\-nmf-n <number>
		Number of NMF components. IF iNMF is used for batch correction, this parameter also sets iNMF number of components. [default: 20]

	-\-knn-K <number>
		Number of nearest neighbors for building kNN graph. [default: 100]

	-\-knn-full-speed
		For the sake of reproducibility, we only run one thread for building kNN indices. Turn on this option will allow multiple threads to be used for index building. However, it will also reduce reproducibility due to the racing between multiple threads.

	-\-kBET
		Calculate kBET.

	-\-kBET-batch <batch>
		kBET batch keyword.

	-\-kBET-alpha <alpha>
		kBET rejection alpha. [default: 0.05]

	-\-kBET-K <K>
		kBET K. [default: 25]

	-\-diffmap
		Calculate diffusion maps.

	-\-diffmap-ndc <number>
		Number of diffusion components. [default: 100]

	-\-diffmap-solver <solver>
		Solver for eigen decomposition, either 'randomized' or 'eigsh'. [default: eigsh]

	-\-diffmap-maxt <max_t>
		Maximum time stamp to search for the knee point. [default: 5000]

	-\-calculate-pseudotime <roots>
		Calculate diffusion-based pseudotimes based on <roots>. <roots> should be a comma-separated list of cell barcodes.

  	-\-louvain
  		Run louvain clustering algorithm.

	-\-louvain-resolution <resolution>
		Resolution parameter for the louvain clustering algorithm. [default: 1.3]

	-\-louvain-class-label <label>
		Louvain cluster label name in result. [default: louvain_labels]

	-\-leiden
		Run leiden clustering algorithm.

	-\-leiden-resolution <resolution>
		Resolution parameter for the leiden clustering algorithm. [default: 1.3]

	-\-leiden-niter <niter>
		Number of iterations of running the Leiden algorithm. If <niter> is negative, run Leiden iteratively until no improvement. [default: -1]

	-\-leiden-class-label <label>
		Leiden cluster label name in result. [default: leiden_labels]

	-\-spectral-louvain
		Run spectral-louvain clustering algorithm.

	-\-spectral-louvain-basis <basis>
		Basis used for KMeans clustering. Can be 'pca' or 'diffmap'. If 'diffmap' is not calculated, use 'pca' instead. [default: diffmap]

	-\-spectral-louvain-nclusters <number>
		Number of first level clusters for Kmeans. [default: 30]

	-\-spectral-louvain-nclusters2 <number>
		Number of second level clusters for Kmeans. [default: 50]

	-\-spectral-louvain-ninit <number>
		Number of Kmeans tries for first level clustering. Default is the same as scikit-learn Kmeans function. [default: 10]

	-\-spectral-louvain-resolution <resolution>.
		Resolution parameter for louvain. [default: 1.3]

	-\-spectral-louvain-class-label <label>
		Spectral-louvain label name in result. [default: spectral_louvain_labels]

	-\-spectral-leiden
		Run spectral-leiden clustering algorithm.

	-\-spectral-leiden-basis <basis>
		Basis used for KMeans clustering. Can be 'pca' or 'diffmap'. If 'diffmap' is not calculated, use 'pca' instead. [default: diffmap]

	-\-spectral-leiden-nclusters <number>
		Number of first level clusters for Kmeans. [default: 30]

	-\-spectral-leiden-nclusters2 <number>
		Number of second level clusters for Kmeans. [default: 50]

	-\-spectral-leiden-ninit <number>
		Number of Kmeans tries for first level clustering. Default is the same as scikit-learn Kmeans function. [default: 10]

	-\-spectral-leiden-resolution <resolution>
		Resolution parameter for leiden. [default: 1.3]

	-\-spectral-leiden-class-label <label>
		Spectral-leiden label name in result. [default: spectral_leiden_labels]

	-\-tsne
		Run FIt-SNE package to compute t-SNE embeddings for visualization.

	-\-tsne-perplexity <perplexity>
		t-SNE's perplexity parameter. [default: 30]

	-\-tsne-initialization <choice>
		<choice> can be either 'random' or 'pca'. 'random' refers to random initialization. 'pca' refers to PCA initialization as described in (CITE Kobak et al. 2019) [default: pca]

  	-\-umap
  		Run umap for visualization.

	-\-umap-K <K>
		K neighbors for umap. [default: 15]

	-\-umap-min-dist <number>
		Umap parameter. [default: 0.5]

	-\-umap-spread <spread>
		Umap parameter. [default: 1.0]

	-\-fle
		Run force-directed layout embedding.

	-\-fle-K <K>
		K neighbors for building graph for FLE. [default: 50]

	-\-fle-target-change-per-node <change>
		Target change per node to stop forceAtlas2. [default: 2.0]

	-\-fle-target-steps <steps>
		Maximum number of iterations before stopping the forceAtlas2 algoritm. [default: 5000]

	-\-fle-memory <memory>
		Memory size in GB for the Java FA2 component. [default: 8]

	-\-net-down-sample-fraction <frac>
		Down sampling fraction for net-related visualization. [default: 0.1]

	-\-net-down-sample-K <K>
		Use <K> neighbors to estimate local density for each data point for down sampling. [default: 25]

	-\-net-down-sample-alpha <alpha>
		Weighted down sample, proportional to radius^alpha. [default: 1.0]

	-\-net-regressor-L2-penalty <value>
		L2 penalty parameter for the deep net regressor. [default: 0.1]

	-\-net-umap
		Run net umap for visualization.

	-\-net-umap-polish-learning-rate <rate>
		After running the deep regressor to predict new coordinate, what is the learning rate to use to polish the coordinates for UMAP. [default: 1.0]

	-\-net-umap-polish-nepochs <nepochs>
		Number of iterations for polishing UMAP run. [default: 40]

	-\-net-umap-out-basis <basis>
		Output basis for net-UMAP. [default: net_umap]

	-\-net-fle
		Run net FLE.

	-\-net-fle-polish-target-steps <steps>
		After running the deep regressor to predict new coordinate, what is the number of force atlas 2 iterations. [default: 1500]

	-\-net-fle-out-basis <basis>
		Output basis for net-FLE. [default: net_fle]

	-\-infer-doublets
		Infer doublets using the method described `here <https://github.com/klarman-cell-observatory/pegasus/raw/master/doublet_detection.pdf>`_. Obs attribute 'doublet_score' stores Scrublet-like doublet scores and attribute 'demux_type' stores 'doublet/singlet' assignments.

 	-\-expected-doublet-rate <rate>
 		The expected doublet rate per sample. By default, calculate the expected rate based on number of cells from the 10x multiplet rate table.

	-\-dbl-cluster-attr <attr>
		<attr> refers to a cluster attribute containing cluster labels (e.g. 'louvain_labels'). Doublet clusters will be marked based on <attr> with the following criteria: passing the Fisher's exact test and having >= 50% of cells identified as doublets. By default, the first computed cluster attribute in the list of leiden, louvain, spectral_ledein and spectral_louvain is used.

	-\-citeseq
	    Input data contain both RNA and CITE-Seq modalities. This will set --focus to be the RNA modality and --append to be the CITE-Seq modality. In addition, 'ADT-' will be added in front of each antibody name to avoid name conflict with genes in the RNA modality.

	-\-citeseq-umap
		For high quality cells kept in the RNA modality, generate a UMAP based on their antibody expression.

	-\-citeseq-umap-exclude <list>
		<list> is a comma-separated list of antibodies to be excluded from the UMAP calculation (e.g. Mouse-IgG1,Mouse-IgG2a).

	\-h, -\-help
		Print out help information.

* Outputs:

	output_name.zarr.zip
		Output file in Zarr format. To load this file in python, use ``import pegasus; data = pegasus.read_input('output_name.zarr.zip')``. The log-normalized expression matrix is stored in ``data.X`` as a CSR-format sparse matrix. The ``obs`` field contains cell related attributes, including clustering results. For example, ``data.obs_names`` records cell barcodes; ``data.obs['Channel']`` records the channel each cell comes from; ``data.obs['n_genes']``, ``data.obs['n_counts']``, and ``data.obs['percent_mito']`` record the number of expressed genes, total UMI count, and mitochondrial rate for each cell respectively; ``data.obs['louvain_labels']`` and ``data.obs['approx_louvain_labels']`` record each cell's cluster labels using different clustring algorithms; ``data.obs['pseudo_time']`` records the inferred pseudotime for each cell. The ``var`` field contains gene related attributes. For example, ``data.var_names`` records gene symbols, ``data.var['gene_ids']`` records Ensembl gene IDs, and ``data.var['selected']`` records selected variable genes. The ``obsm`` field records embedding coordiates. For example, ``data.obsm['X_pca']`` records PCA coordinates, ``data.obsm['X_tsne']`` records tSNE coordinates, ``data.obsm['X_umap']`` records UMAP coordinates, ``data.obsm['X_diffmap']`` records diffusion map coordinates, and ``data.obsm['X_fle']`` records the force-directed layout coordinates from the diffusion components. The ``uns`` field stores other related information, such as reference genome (``data.uns['genome']``). This file can be loaded into R and converted into a Seurat object.

	output_name.<group>.h5ad
		Optional output. Only exists if '--output-h5ad' is set. Results in h5ad format per focused <group>. This file can be loaded into R and converted into a Seurat object.

	output_name.<group>.loom
		Optional output. Only exists if '--output-loom' is set. Results in loom format per focused <group>.

	output_name.<group>.filt.xlsx
		 Optional output. Only exists if '--output-filtration-results' is set. Filtration statistics per focused <group>. This file has two sheets --- Cell filtration stats and Gene filtration stats. The first sheet records cell filtering results and it has 10 columns: Channel, channel name; kept, number of cells kept; median_n_genes, median number of expressed genes in kept cells; median_n_umis, median number of UMIs in kept cells; median_percent_mito, median mitochondrial rate as UMIs between mitochondrial genes and all genes in kept cells; filt, number of cells filtered out; total, total number of cells before filtration, if the input contain all barcodes, this number is the cells left after '--min-genes-on-raw' filtration; median_n_genes_before, median expressed genes per cell before filtration; median_n_umis_before, median UMIs per cell before filtration; median_percent_mito_before, median mitochondrial rate per cell before filtration. The channels are sorted in ascending order with respect to the number of kept cells per channel. The second sheet records genes that failed to pass the filtering. This sheet has 3 columns: gene, gene name; n_cells, number of cells this gene is expressed; percent_cells, the fraction of cells this gene is expressed. Genes are ranked in ascending order according to number of cells the gene is expressed. Note that only genes not expressed in any cell are removed from the data. Other filtered genes are marked as non-robust and not used for TPM-like normalization.

	output_name.<group>.filt.gene.pdf
		Optional output. Only exists if '--plot-filtration-results' is set. This file contains violin plots contrasting gene count distributions before and after filtration per channel per focused <group>.

	output_name.<group>.filt.UMI.pdf
		Optional output. Only exists if '--plot-filtration-results' is set. This file contains violin plots contrasting UMI count distributions before and after filtration per channel per focused <group>.

	output_name.<group>.filt.mito.pdf
		Optional output. Only exists if '--plot-filtration-results' is set. This file contains violin plots contrasting mitochondrial rate distributions before and after filtration per channel per focused <group>.

	output_name.<group>.hvf.pdf
		Optional output. Only exists if '--plot-hvf' is set. This file contains a scatter plot describing the highly variable gene selection procedure per focused <group>.

	output_name.<group>.<channel>.dbl.png
		Optional output. Only exists if '--infer-doublets' is set. Each figure consists of 4 panels showing diagnostic plots for doublet inference. If there is only one channel in <group>, file name becomes output_name.<group>.dbl.png.

* Examples::

	pegasus cluster -p 20 --correct-batch-effect --louvain --tsne example_10x.h5 example_out
	pegasus cluster -p 20 --leiden --umap --net-fle example.zarr.zip example_out


---------------------------------


``pegasus de_analysis``
^^^^^^^^^^^^^^^^^^^^^^^^

Once we have the clusters, we can detect markers using ``pegasus de_analysis``. We will calculate Mann-Whitney U test and AUROC values by default.

Type::

	pegasus de_analysis -h

to see the usage information::

	Usage:
		pegasus de_analysis [options] (--labels <attr>) <input_data_file> <output_spreadsheet>
		pegasus de_analysis -h

* Arguments:

	input_data_file
		Single cell data with clustering calculated. DE results would be written back.

	output_spreadsheet
		Output spreadsheet with DE results.

* Options:

	-\-labels <attr>
		<attr> used as cluster labels. [default: louvain_labels]

	\-p <threads>
		Use <threads> threads. [default: 1]

	-\-de-key <key>
		Store DE results into AnnData varm with key = <key>. [default: de_res]

	-\-t
		Calculate Welch's t-test.

	-\-fisher
		Calculate Fisher's exact test.

	-\-temp-folder <temp_folder>
		Joblib temporary folder for memmapping numpy arrays.

	-\-alpha <alpha>
		Control false discovery rate at <alpha>. [default: 0.05]

	-\-ndigits <ndigits>
		Round non p-values and q-values to <ndigits> after decimal point in the excel. [default: 3]

	-\-quiet
		Do not show detailed intermediate outputs.

	\-h, -\-help
		Print out help information.

* Outputs:

	input_data_file
		DE results would be written back to the 'varm' field with name set by '--de-key <key>'.

	output_spreadsheet
		An excel spreadsheet containing DE results. Each cluster has two tabs in the spreadsheet. One is for up-regulated genes and the other is for down-regulated genes.
		If DE was performed on conditions within each cluster. Each cluster will have number of conditions tabs and each condition tab contains two spreadsheet: up for up-regulated genes and down for down-regulated genes.

* Examples::

	pegasus de_analysis -p 26 --labels louvain_labels --t --fisher example.zarr.zip example_de.xlsx


---------------------------------


``pegasus find_markers``
^^^^^^^^^^^^^^^^^^^^^^^^

Once we have the DE results, we can optionally find cluster-specific markers with gradient boosting using ``pegasus find_markers``.

Type::

	pegasus find_markers -h

to see the usage information::

	Usage:
		pegasus find_markers [options] <input_data_file> <output_spreadsheet>
		pegasus find_markers -h

* Arguments:

	input_h5ad_file
		Single cell data after running the de_analysis.

	output_spreadsheet
		Output spreadsheet with LightGBM detected markers.

* Options:

	\-p <threads>
		Use <threads> threads. [default: 1]

	-\-labels <attr>
		<attr> used as cluster labels. [default: louvain_labels]

	-\-de-key <key>
		Key for storing DE results in 'varm' field. [default: de_res]

	-\-remove-ribo
		Remove ribosomal genes with either RPL or RPS as prefixes.

	-\-min-gain <gain>
		Only report genes with a feature importance score (in gain) of at least <gain>. [default: 1.0]

	-\-random-state <seed>
		Random state for initializing LightGBM and KMeans. [default: 0]



	\-h, -\-help
		Print out help information.

* Outputs:

	output_spreadsheet
		An excel spreadsheet containing detected markers. Each cluster has one tab in the spreadsheet and each tab has six columns, listing markers that are strongly up-regulated, weakly up-regulated, down-regulated and their associated LightGBM gains.

* Examples::

	pegasus find_markers --labels louvain_labels --remove-ribo --min-gain 10.0 -p 10 example.zarr.zip example.markers.xlsx


---------------------------------


``pegasus annotate_cluster``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Once we have the DE results, we could optionally identify putative cell types for each cluster using ``pegasus annotate_cluster``.
This command has two forms: the first form generates putative annotations, and the second form write annotations into the Zarr object.

Type::

	pegasus annotate_cluster -h

to see the usage information::

	Usage:
		pegasus annotate_cluster [--marker-file <file> --de-test <test> --de-alpha <alpha> --de-key <key> --minimum-report-score <score> --do-not-use-non-de-genes] <input_data_file> <output_file>
		pegasus annotate_cluster --annotation <annotation_string> <input_data_file>
		pegasus annotate_cluster -h

* Arguments:

	input_data_file
		Single cell data with DE analysis done by ``pegasus de_analysis``.

	output_file
		Output annotation file.

* Options:

	-\-markers <str>
		<str> is a comma-separated list. Each element in the list either refers to a JSON file containing legacy markers, or 'human_immune'/'mouse_immune'/'human_brain'/'mouse_brain'/'human_lung' for predefined markers. [default: human_immune]

	-\-de-test <test>
		DE test to use to infer cell types. [default: mwu]

	-\-de-alpha <alpha>
		False discovery rate to control family-wise error rate. [default: 0.05]

	-\-de-key <key>
		Keyword where the DE results store in 'varm' field. [default: de_res]

	-\-minimum-report-score <score>
		Minimum cell type score to report a potential cell type. [default: 0.5]

	-\-do-not-use-non-de-genes
		Do not count non DE genes as down-regulated.

	-\-annotation <annotation_string>
		Write cell type annotations in <annotation_string> into <input_data_file>. <annotation_string> has this format: ``'anno_name:clust_name:anno_1;anno_2;...;anno_n'``,
		where ``anno_name`` is the annotation attribute in the Zarr object, ``clust_name`` is the attribute with cluster ids, and ``anno_i`` is the annotation for cluster i.

	\-h, -\-help
		Print out help information.

* Outputs:

	output_file
		This is a text file. For each cluster, all its putative cell types are listed in descending order of the cell type score. For each putative cell type, all markers support this cell type are listed. If one putative cell type has cell subtypes, all subtypes will be listed under this cell type.

* Examples::

	pegasus annotate_cluster example.zarr.zip example.anno.txt
	pegasus annotate_cluster --markers human_immune,human_lung lung.zarr.zip lung.anno.txt
	pegasus annotate_cluster --annotation "anno:louvain_labels:T cells;B cells;NK cells;Monocytes" example.zarr.zip


---------------------------------



``pegasus plot``
^^^^^^^^^^^^^^^^^

We can make a variety of figures using ``pegasus plot``.

Type::

	pegasus plot -h

to see the usage information::

	Usage:
  		pegasus plot [options] [--restriction <restriction>...] [--palette <palette>...] <plot_type> <input_file> <output_file>
		pegasus plot -h

* Arguments:

	plot_type
		Plot type, either 'scatter' for scatter plots, 'compo' for composition plots, or 'wordcloud' for word cloud plots.

	input_file
		Single cell data in Zarr or H5ad format.

  	output_file
  		Output image file.

* Options:

	-\-dpi <dpi>
		DPI value for the figure. [default: 500]

	-\-basis <basis>
		Basis for 2D plotting, chosen from 'tsne', 'fitsne', 'umap', 'pca', 'fle', 'net_tsne', 'net_umap' or 'net_fle'. [default: umap]

	-\-attributes <attrs>
		<attrs> is a comma-separated list of attributes to color the basis. This option is only used in 'scatter'.

	-\-restriction <restriction>...
		Set restriction if you only want to plot a subset of data. Multiple <restriction> strings are allowed. Each <restriction> takes the format of 'attr:value,value', or 'attr:~value,value..' which means excluding values. This option is used in 'composition' and 'scatter'.

	-\-alpha <alpha>
		Point transparent parameter. Can be a single value or a list of values separated by comma used for each attribute in <attrs>.

	-\-legend-loc <str>
		Legend location, can be either "right margin" or "on data". If a list is provided, set 'legend_loc' for each attribute in 'attrs' separately. [default: "right margin"]

	-\-palette <str>
		Used for setting colors for every categories in categorical attributes. Multiple <palette> strings are allowed. Each string takes the format of 'attr:color1,color2,...,colorn'. 'attr' is the categorical attribute and 'color1' - 'colorn' are the colors for each category in 'attr' (e.g. 'cluster_labels:black,blue,red,...,yellow'). If there is only one categorical attribute in 'attrs', ``palletes`` can be set as a single string and the 'attr' keyword can be omitted (e.g. "blue,yellow,red").

	-\-show-background
		Show points that are not selected as gray.

	-\-nrows <nrows>
		Number of rows in the figure. If not set, pegasus will figure it out automatically.

	-\-ncols <ncols>
		Number of columns in the figure. If not set, pegasus will figure it out automatically.

	-\-panel-size <sizes>
		Panel size in inches, w x h, separated by comma. Note that margins are not counted in the sizes. For composition, default is (6, 4). For scatter plots, default is (4, 4).

	-\-left <left>
		Figure's left margin in fraction with respect to panel width.

	-\-bottom <bottom>
		Figure's bottom margin in fraction with respect to panel height.

	-\-wspace <wspace>
		Horizontal space between panels in fraction with respect to panel width.

	-\-hspace <hspace>
		Vertical space between panels in fraction with respect to panel height.

	-\-groupby <attr>
		Use <attr> to categorize the cells for the composition plot, e.g. cell type.

	-\-condition <attr>
		Use <attr> to calculate frequency within each category defined by '--groupby' for the composition plot, e.g. donor.

	-\-style <style>
		Composition plot styles. Can be either 'frequency' or 'normalized'. [default: normalized]

	-\-factor <factor>
		Factor index (column index in data.uns['W']) to be used to generate word cloud plot.

	-\-max-words <max_words>
		Maximum number of genes to show in the image. [default: 20]

	\-h, -\-help
		Print out help information.

Examples::

	pegasus plot scatter --basis tsne --attributes louvain_labels,Donor example.h5ad scatter.pdf
	pegasus plot compo --groupby louvain_labels --condition Donor example.zarr.zip compo.pdf
	pegasus plot wordcloud --factor 0 example.zarr.zip word_cloud_0.pdf


---------------------------------

``pegasus scp_output``
^^^^^^^^^^^^^^^^^^^^^^^

If we want to visualize analysis results on `single cell portal <https://singlecell.broadinstitute.org/single_cell>`_ (SCP), we can generate required files for SCP using this subcommand.

Type::

	pegasus scp_output -h

to see the usage information::

	Usage:
		pegasus scp_output <input_data_file> <output_name>
		pegasus scp_output -h

* Arguments:

	input_data_file
		Analyzed single cell data in zarr format.

	output_name
		Name prefix for all outputted files.

* Options:

	-\-dense
		Output dense expression matrix instead.

	-\-round-to <ndigit>
		Round expression to <ndigit> after the decimal point. [default: 2]

	\-h, -\-help
		Print out help information.

* Outputs:

	output_name.scp.metadata.txt, output_name.scp.barcodes.tsv, output_name.scp.genes.tsv, output_name.scp.matrix.mtx, output_name.scp.*.coords.txt, output_name.scp.expr.txt
		Files that single cell portal needs.

* Examples::

	pegasus scp_output example.zarr.zip example


---------------------------------

``pegasus check_indexes``
^^^^^^^^^^^^^^^^^^^^^^^^^

If we run CITE-Seq or any kind of hashing, we need to make sure that the library indexes of CITE-Seq/hashing do not collide with 10x's RNA indexes. This command can help us to determine which 10x index sets we should use.

Type::

	pegasus check_indexes -h

to see the usage information::

	Usage:
		pegasus check_indexes [--num-mismatch <mismatch> --num-report <report>] <index_file>
		pegasus check_indexes -h

* Arguments:

	index_file
		Index file containing CITE-Seq/hashing index sequences. One sequence per line.

* Options:

	-\-num-mismatch <mismatch>
		Number of mismatch allowed for each index sequence. [default: 1]

  	-\-num-report <report>
  		Number of valid 10x indexes to report. Default is to report all valid indexes. [default: 9999]

  	\-h, -\-help
  		Print out help information.

* Outputs:

	Up to <report> number of valid 10x indexes will be printed out to standard output.

* Examples::

	pegasus check_indexes --num-report 8 index_file.txt
