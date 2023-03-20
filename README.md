# PyTorch implementation of Jan Chorowski, Jan 2019 paper"

Others ressources :
https://github.com/vincentherrmann/pytorch-wavenet

This is a PyTorch implementation of https://arxiv.org/abs/1901.08810.

[Under Construction]

## Update June 14, 2020

Training a simpler model to perform the "mfcc inversion" task.  The idea is:

1. preprocess:  wav -> mfcc
2. vqvae: mfcc -> z -> mfcc
3. mfcc-inverter: mfcc -> wav

The mfcc-inverter model is just a wavenet conditioned on mfcc vectors (1 every
160 timesteps) which produces the original wav used to compute the mfcc
vectors.  It is a probabilistic inverse of the preprocessing step.

It should be noted that there is loss of information in the preprocessing step,
so the inverter cannot attain 100% accuracy unless it overfits the data.

Once the mfcc inverter model is trained, it can be used in conjunction with
a vq-vae model that starts and ends with MFCC.  One advantage to this is that
the training of the vq-vae model may be slightly less compute intensive, since
there are only 39 components (one mfcc vector plus first and second
derivatives) every 160 timesteps, instead of 160.

See results directory for some preliminary training results.

## Update April 14, 2019

Began training on Librispeech dev (http://www.openslr.org/resources/12/dev-clean.tar.gz),
see dat/example\_train.log

## Update May 12, 2019

First runs using vqvae mode.  After ~200 iterations, only one quantized vector is
used as a representative.  Currently troubleshooting.

## Update Nov 7, 2019

Resumed work as of Sept, 2019.  Implemented EMA for updates.  Fixed a bug in
the VQVAE Loss function in vq_bn.py:312

* Was: l2_loss_embeds = self.l2(self.bn.ze, self.bn.emb)
* Now: l2_loss_embeds = self.l2(self.bn.sg(self.bn.ze), self.bn.emb)

Training still exhibits codebook collapse.  This seems due to the phenomenon of
WaveNet learning to rely exclusively on the autoregressive input and ignore the
conditioning input.


# TODO
1. VAE and VQVAE versions of the bottleneck / training objectives [DONE]
2. Inference mode
 
# Example training setup

```sh
code_dir=/path/to/ae-wavenet
run_dir=/path/to/my_runs

# Get the data
cd $run_dir
wget http://www.openslr.org/resources/12/dev-clean.tar.gz
tar zxvf dev-clean.tar.gz
$code_dir/scripts/librispeech_to_rdb.sh LibriSpeech/dev-clean > librispeech.dev-clean.rdb 

# Preprocess the data
# This stores a flattened, indexed copy of the sound data plus calculated MFCCs
python preprocess.py librispeech.dev-clean.rdb librispeech.dev-clean.dat -nq 256 -sr 16000

# Train
# New mode
cd $code_dir
python train.py new -af par/arch.basic.json -tf par/train.basic.json -nb 4 -si 1000 \
  -vqn 1000 $run_dir/model%.ckpt $run_dir/librispeech.dev-clean.dat $run_dir/data_slices.dat

# Resume mode - resume from step 10000, save every 1000 steps
python train.py resume -nb 4 -si 1000 $run_dir/model%.ckpt $run_dir/model10000.ckpt

```

