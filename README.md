# CTC segmentation

<!-- Badges -->
[![version](https://img.shields.io/pypi/v/ctc-segmentation?style=plastic)](https://pypi.org/project/ctc-segmentation/)
[![downloads](https://img.shields.io/pypi/dm/ctc-segmentation?style=plastic)](https://pypi.org/project/ctc-segmentation/)
[![build status](https://github.com/lumaku/ctc-segmentation/actions/workflows/python-package.yml/badge.svg)](https://github.com/lumaku/ctc-segmentation/actions/workflows/python-package.yml)

CTC segmentation can be used to find utterances alignments within large audio files.

* This repository contains the `ctc-segmentation` python package.
* A description of the algorithm is in https://arxiv.org/abs/2007.09127
* The code used in the paper is archived in https://github.com/cornerfarmer/ctc_segmentation


# Usage

The CTC segmentation package is not standalone, as it needs a neural network with CTC output. It is integrated in these frameworks:

* In ESPnet 1 as corpus recipe: [Alignment script](https://github.com/espnet/espnet/blob/master/espnet/bin/asr_align.py), [Example recipe](https://github.com/espnet/espnet/tree/master/egs/tedlium2/align1), [Demo](https://github.com/espnet/espnet#ctc-segmentation-demo )
* In ESPnet 2, as script or directly as python interface: [Alignment script](https://github.com/lumaku/espnet/blob/espnet2_ctc_segmentation/espnet2/bin/asr_align.py), [Demo](https://github.com/lumaku/espnet/tree/espnet2_ctc_segmentation#ctc-segmentation-demo )
* In Nvidia NeMo as dataset creation tool: [Documentation](https://docs.nvidia.com/deeplearning/nemo/user-guide/docs/en/main/tools/ctc_segmentation.html), [Example](https://github.com/NVIDIA/NeMo/blob/main/tutorials/tools/CTC_Segmentation_Tutorial.ipynb) 



# Installation

* With `pip`:
```sh
pip install ctc-segmentation
```

* From the Arch Linux AUR as `python-ctc-segmentation-git` using your favourite AUR helper.

* From source:
```sh
git clone https://github.com/lumaku/ctc-segmentation
cd ctc-segmentation
cythonize -3 ctc_segmentation/ctc_segmentation_dyn.pyx
python setup.py build
python setup.py install --optimize=1 --skip-build
```


# Toolkit Integration

CTC segmentation requires CTC activations of an already trained CTC-based network. Example code can be found in the alignment scripts `asr_align.py` of ESPnet 1 or ESpnet 2.

### Steps to alignment for regular ASR

1. `prepare_text` filters characters not in the dictionary, and generates the character matrix.
2. `ctc_segmentation` computes character-wise alignments from the CTC log posterior probabilites.
3. `determine_utterance_segments` converts char-wise alignments to utterance-wise alignments.
4. In a post-processing step, segments may be filtered by their confidence value.


### Steps to alignment for different use-cases

Sometimes the ground truth data is not text, but a sequence of tokens, or a list of protein segments.

In this case, replace the function `prepare_text` with a function that suits better for your data. For this, produce a numpy matrix, e.g. a shape `[L,1]` with L as total ground truth length including separator symbols (`ground_truth_mat`) and also, a list that gives the starting indices of the separate segments (`utt_begin_indices`).

Examples: See `prepare_tokenized_text` in `ctc_segmentation.py`, or the example included in the NeMo toolkit.


### Segments clean-up

Segments that were written to a `segments` file can be filtered using the confidence score. This is the minium confidence score in log space as described in the paper. 

Utterances with a low confidence score are discarded in a data clean-up. This parameter may need adjustment depending on dataset, ASR model and language. For the German ASR model, a value of -1.5 worked very well; for TEDlium, a lower value of about -5.0 seemed more practical.

```bash
awk -v ms=${min_confidence_score} '{ if ($5 > ms) {print} }' ${unfiltered} > ${filtered}
```


# Parameters

There are several notable parameters to adjust the working of the algorithm that can be found in the class `CtcSegmentationParameters`:


### Data preparation parameters

* Localization: The character set is taken from the model dict, i.e., usually are generated with SentencePiece. An ASR model trained in the corresponding language and character set is needed. For asian languages, no changes to the CTC segmentation parameters should be necessary. One exception: If the character set contains any punctuation characters, "#", or the Greek char "ε", adapt the setting in an instance of `CtcSegmentationParameters` in `segmentation.py`.

* `CtcSegmentationParameters` includes a blank character. Copy over the Blank character from the dictionary to the configuration, if in the model dictionary e.g. "\<blank>" instead of the default "_" is used. If the Blank in the configuration and in the dictionary mismatch, the algorithm raises an IndexError at backtracking.

* If `replace_spaces_with_blanks` is True, then spaces in the ground truth sequence are replaces by blanks. This option is enabled by default and improves compability with dictionaries with unknown space characters.

### Alignment parameters

* `min_window_size`: Minimum window size considered for a single utterance. The current default value should be OK in most cases.

* To align utterances with longer unkown audio sections between them, use `blank_transition_cost_zero` (default: False). With this option, the stay transition in the blank state is free. A transition to the next character is only consumed if the probability to switch is higher. In this way, more time steps can be skipped between utterances. Caution: in combination with `replace_spaces_with_blanks == True`, this may lead to misaligned segments.

### Time stamp parameters

Directly set the parameter `index_duration` to give the corresponding time duration of one CTC output index (in seconds).

**Example:** For a given sample rate, say, 16kHz, `fs=16000`. Then, how many sample points correspond to one ctc output index? In some ASR systems, this can be calculated from the hop length of the windowing times encoder subsampling factor. For example, if the hop length of the frontend windowing is 128, and the subsampling factor in the encoder is 4, totalling 512 sample points for one CTC index. Then `index_duration = 512 / 16000`.

**Note:** In earlier versions, `index_duration` was not used and the time stamps were determined from the values of `subsampling_factor` and `frame_duration_ms`. To derive `index_duration` from these values, calculate`frame_duration_ms * subsampling_factor / 1000`.


# How it works

### 1. Forward propagation

Character probabilites from each time step are obtained from a CTC-based network.
With these, transition probabilities are mapped into a trellis diagram.
To account for preambles or unrelated segments in audio files, the transition cost are set to zero for the start-of-sentence or blank token.

![Forward trellis](doc/1_forward.png)

### 2. Backtracking

Starting from the time step with the highest probability for the last character, backtracking determines the most probable path of characters through all time steps.

![Backward path](doc/2_backtracking.png)

### 3. Confidence score

As this method generates a probability for each aligned character, a confidence score for each utterance can be derived.
For example, if a word within an utterance is missing, this value is low.

![Confidence score](doc/3_scoring.png)

The confidence score helps to detect and filter-out bad utterances.


# Reference

The full paper can be found in the preprint https://arxiv.org/abs/2007.09127 or published at https://doi.org/10.1007/978-3-030-60276-5_27. To cite this work:

```
@InProceedings{ctcsegmentation,
author="K{\"u}rzinger, Ludwig
and Winkelbauer, Dominik
and Li, Lujun
and Watzel, Tobias
and Rigoll, Gerhard",
editor="Karpov, Alexey
and Potapova, Rodmonga",
title="CTC-Segmentation of Large Corpora for German End-to-End Speech Recognition",
booktitle="Speech and Computer",
year="2020",
publisher="Springer International Publishing",
address="Cham",
pages="267--278",
abstract="Recent end-to-end Automatic Speech Recognition (ASR) systems demonstrated the ability to outperform conventional hybrid DNN/HMM ASR. Aside from architectural improvements in those systems, those models grew in terms of depth, parameters and model capacity. However, these models also require more training data to achieve comparable performance.",
isbn="978-3-030-60276-5"
}
```
