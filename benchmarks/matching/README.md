# Image Matching Benchmark

The *Image Matching Benchmark* (IMB) evaluates the ability of a patch descriptor to match a patch in a reference image to its corresponding patch in a target image. It is called "image matching" because matching is restricted to patches coming from two images only, a reference and a target.

This problem is formulated as a ranking task: each reference patch is compared to all target patches by computing the corresponding descriptor dissimilarity scores. The resulting reference-target patch pairs are then entered together with their scores in a patch pair rank list and the quality of the latter is assessed by computing its average precision (AP). Finally, the AP values for a number of reference-target image pairs are averaged to produce the mean average precision (mAP) of the descriptor. 

Two types of scores are considered: the raw dissimilarity scores and the same scores, but renormalized by dividing them, for each reference patch, by the dissimilarity of the second nearest target patch. This protocol is similar to the nearest neighbor patch classifier evaluation explored in [1].

[TOC]

## Benchmark definitions

The benchmark is composed of a number of tasks, each defined by a corresponding `.benchmark` file. There are four such files:

```bash
> ls -1 benchmarks/matching/*benchmark
train_easy_illum.benchmark
train_easy_viewpoint.benchmark
train_hard_illum.benchmark
train_hard_viewpoint.benchmark
```

Once the test set will be made available, corresponding `test_*` files will be provided. 

The corresponding tasks are as follows:

* `train_easy_illum.benchmark` contains pairs of images with easy affine jitters between the corresponding patches from scenes where the illumination changes.
* `train_easy_viewpoint.benchmark` contains pairs of images with easy affine jitter between the patches from scenes where the viewpoint changes.
* Similarly, `train_hard_illum.benchmark` and `train_hard_viewpoint.benchmark` contains patches with harder affine jitter between the patches.

### Benchmark file format

The content of each `*.benchmark` file is of the type:

```
im_a,im_b
im_x,im_y
...
```

where each line specifies a pair of patch-images among which the descriptors should be matched. [Recall](../../README.md#reading-patches) that patches in *HPatches* are organized in patch-images, each identified by a pair `SEQNAME.IMNAME`. 

For exxample, the content of `train_easy_illum` looks as follows:

```bash
> cat benchmarks/matching/train_easy_illum.benchmark
i_boutique.ref,i_boutique.e1
i_boutique.ref,i_boutique.e2
...
```

This means that all patches in the patch-image `i_botique.ref` should be matched to all patches in the patch-image `i_botique.e1`. The format of the patch-images and their identifiers is discussed [here](../../README.md#reading-patches).

## Entering the benchmarks

Entering the benchmark is conceptually simple. One needs to:

1. Identify an image pairs to be matched.
2. Use a descriptor to compare each patch in the reference image (first image in the pair) to each patch in the target mage (second image in the pair). For this step, one can use the preferred distance measure (e.g. L1 or L2) or any other dissimilarity score.
3. Write the results of such comparisons to a ranked list.

In more detail, for each `*.benchmark` file, one needs to write a corresponding `*.results` file. In order to allow comparing different descriptors, each file must be store in a descriptor-specific directory. So, if `my_desc` is the name of your descriptor, you need to write the four files:

```
results/matching/my_desc/test_easy_illum.results
results/matching/my_desc/test_easy_viewpoint.results
results/matching/my_desc/test_hard_illum.results
results/matching/my_desc/test_hard_viewpoint.results
```

### Result file format

A result file is organized as follows:

```
First patch-image pair: reference image, target image
  For each reference patch, the index of the corresponding nearest target patch
  Corresponding distances
  For each reference patch, the index of the corresponding 2-nd nearest target patch
  Corresponding distances
  ...
Second patch-image pair: reference image, target image
  For each reference patch, the index of the corresponding nearest target patch
  Corresponding distances
  ...
```

For example, the file `train_easy_illum.results` may look something like:

```bash
> cat results/my_desc/matching/train_easy_illum.results 
i_boutique.ref,i_boutique.e1
  732,          761,          154,          564,          ...
  7.174843e+00, 1.438751e+01, 6.510703e+00, 1.225562e+01, ...
  0,            828,          632,          95,           ...
  1.310076e+01, 1.514586e+01, 8.786040e+00, 1.297130e+01, ...
  ...
i_boutique.ref,i_boutique.e2
  0,            80,           400,          3,            ...
  1.503223e+01, 1.191290e+01, 8.384595e+00, 6.479039e+00, ...
  ...
```

> **Remark:** by construction, the dissimilarity value should increase along each column.

We can define this file more formally as follows. Let `im_a` and `im_b` be the identifiers of two patch-images.  Let `nn(im_a.idx,im_b,n)` be the *index* of the *n*-th nearest neighbour of a patch `im_a.idx` to the patches in image `im_b`. Furthermore, let ``ds(im_a.idx,im_b,n)`` be the corresponding dissimilarity value. Then the file content is as follows:

```
im_a,im_b
nn(im_a.0,im_b,1), nn(im_a.1,im_b,1), ..., nn(im_a.M,im_b,1)
ds(im_a.0,im_b,1), ds(im_a.1,im_b,1), ..., ds(im_a.M,im_b,1)
nn(im_a.0,im_b,2), nn(im_a.1,im_b,2), ..., nn(im_a.M,im_b,2)
ds(im_a.0,im_b,2), ds(im_a.1,im_b,2), ..., ds(im_a.M,im_b,2)
...
nn(im_a.0,im_b,K), nn(im_a.K,im_b,K), ..., nn(im_a.M,im_b,N)
ds(im_a.0,im_b,K), ds(im_a.K,im_b,K), ..., ds(im_a.M,im_b,N)
im_c,im_d
nn(im_c.0,im_d,1), nn(im_c.1,im_d,1), ..., nn(im_c.P,im_d,1)
ds(im_c.0,im_d,1), ds(im_c.1,im_d,1), ..., ds(im_c.P,im_d,1)
...
```

For example, if a sequence `s_boring` contains only two patches,
and the nearest neighbors of `s_boring.a.0` are `s_boring.b.1` and `s_boring.b.0` with distance 12.3 and 14.2 respectively, and `s_boring.b.0` and  `s_boring.b.1` are the nearest neighboors to `s_boring.a.1` with distances 7.5 and 27.4, the results file should look something like:

```
s_boring.a,s_boring.b
1, 0
12.3, 7.5
0, 1
14.2, 27.4
```

### Generating and validating the result files

You can generate the results files with MATLAB scripts `matching_compute.m` and compute the PR curves and the AP with `matching_eval.m` in the *HBench* toolbox.

## Ground-truth labels

For each `*.benchmark` file, there is a corresponding `*.labels` file containing the ground-truth matching patches for each query patch. For example, the ground-truth file for the benchmark `train_easy_illum` looks like this:

```bash
> cat benchmarks/matching/train_easy_illum.labels 
i_boutique.ref,i_boutique.e1
0,1,2,3,4,5,6,...
0,1,2,3,4,5,6,...
i_boutique.ref,i_boutique.e2
0,1,2,3,4,5,6,...
0,1,2,3,4,5,6,...
```

Here each image pairs has to lines. The first line is a list of patch indexes in the first image in the pair and the second line is a list of the *corresponding* patch indexes in the second image. In this example, indexes happen to be progressive due to the dataset construction method, but they need not to be in general.

Using this information, it is possible to compute the ROC and PR curves and mAP performance for the image matching task.

## References

[1] K. Mikolajczyk and C. Schmid. A performance evaluation of local descriptors. In IEEE TPAMI 2005.
